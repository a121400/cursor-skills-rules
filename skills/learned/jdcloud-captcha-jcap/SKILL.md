---
name: jdcloud-captcha-jcap
description: 京东云验证码(JCAP)滑块逆向与纯协议求解。WASM jcap.js 桥接、fp/check 接口、tk/ct 加密拼接、ddddocr 缺口检测、vt token 获取。适用于京东系及使用 JCAP 的第三方站点（crxn.cn 宜宾等）。当目标站点使用 captcha-api-global.jdcloud.com、x-jdcloud-captcha-auth header、jcap.js WASM 模块时使用。
---

# 京东云验证码 (JCAP) 滑块逆向

## 识别特征

遇到以下任一特征即可确认为 JCAP：

| 特征 | 示例 |
|------|------|
| API 域名 | `captcha-api-global.jdcloud.com` |
| 请求头 | `x-jdcloud-captcha-auth: ...` |
| WASM 模块 | JS 中加载 `jcap.js`（含 `getCaptchaAuth`、`getEncryptData` 导出） |
| 接口路径 | `/cgi-bin/api/fp`、`/cgi-bin/api/check` |
| 参数 | `si`(sessionId)、`tk`(加密轨迹)、`ct`(加密指纹)、`st`(state token) |
| 响应 | `tp=9`(无感通过)、`tp=4`(降级滑块)、`vt`(verify token) |

## 完整流程

```
1. getSessionId     → 获取 si (sessionId)
2. fp (POST)        → 上报设备指纹，获取 st (state token)
3. check (POST)     → 第一次提交触发 tp=4 降级，获取滑块图片 (b1=背景, b2=滑块)
4. 缺口检测         → ddddocr.slide_match 识别缺口 x 坐标
5. check (POST)     → 提交滑动轨迹，通过后返回 vt (verify token)
6. 业务接口         → 带 vt + si 调用目标 API
```

## 核心架构：WASM Bridge

JCAP 的 `tk` 和 `ct` 参数由 WASM 加密生成，无法纯 Python 复现。**必须**通过 Node.js 子进程桥接调用原始 `jcap.js`。

### jcap.js 获取

从目标站点的 Network 抓包中搜索 `jcap` 或 `getCaptchaAuth`，下载对应的 JS 文件。该文件内嵌 WASM 二进制。

### crypto_bridge.js (Node.js 桥接)

```javascript
const fs = require('fs');
const path = require('path');
const readline = require('readline');

let wasmModule = null;

function loadJcap() {
    return new Promise((resolve, reject) => {
        // 补浏览器环境
        global.window = global;
        global.document = { createElement: () => ({}), head: { appendChild: () => {} } };
        global.navigator = { connection: {}, userAgent: 'Mozilla/5.0' };
        global.XMLHttpRequest = class { open() {} send() {} };

        const code = fs.readFileSync(path.join(__dirname, 'jcap.js'), 'utf8');
        const wrappedCode = `
            var Module = {};
            Module.onRuntimeInitialized = function() { global._wasmReady = true; };
            ${code}
            global._jcapModule = typeof f !== 'undefined' ? f : Module;
        `;
        try { eval(wrappedCode); } catch (e) { /* ignore browser-specific errors */ }

        const checkReady = setInterval(() => {
            if (global._wasmReady || (global.window?.f?.getCaptchaAuth)) {
                clearInterval(checkReady);
                wasmModule = global.window.f || global._jcapModule;
                resolve();
            }
        }, 100);
        setTimeout(() => { clearInterval(checkReady); reject(new Error('WASM timeout')); }, 10000);
    });
}

async function main() {
    await loadJcap();
    const rl = readline.createInterface({ input: process.stdin });
    rl.on('line', (line) => {
        try {
            const cmd = JSON.parse(line);
            let result;
            if (cmd.action === 'getCaptchaAuth') {
                result = wasmModule.getCaptchaAuth(cmd.appId);
            } else if (cmd.action === 'getEncryptData') {
                const enc = wasmModule.getEncryptData(cmd.data, cmd.key);
                result = typeof enc === 'object' ? JSON.stringify(enc) : enc;
            }
            process.stdout.write(JSON.stringify({ ok: true, result }) + '\n');
        } catch (e) {
            process.stdout.write(JSON.stringify({ ok: false, error: e.message }) + '\n');
        }
    });
    process.stdout.write(JSON.stringify({ ok: true, result: 'ready' }) + '\n');
}
main();
```

### Python WasmBridge 类

```python
import subprocess, json
from pathlib import Path

class WasmBridge:
    def __init__(self):
        js_path = Path(__file__).resolve().parent / "crypto_bridge.js"
        self.proc = subprocess.Popen(
            ["node", str(js_path)],
            stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
            cwd=str(js_path.parent),
        )
        self.proc.stdout.readline()  # wait for "ready"

    def call(self, action, **kw):
        if self.proc.poll() is not None:
            raise RuntimeError("WASM bridge exited")
        payload = json.dumps({"action": action, **kw}) + "\n"
        self.proc.stdin.write(payload.encode())
        self.proc.stdin.flush()
        line = self.proc.stdout.readline().decode(errors="ignore").strip()
        return json.loads(line)

    def auth(self, app_id):
        """生成 x-jdcloud-captcha-auth header"""
        return self.call("getCaptchaAuth", appId=str(app_id))["result"]

    def encrypt(self, data, key):
        """WASM 加密数据，用于 tk/ct 参数"""
        return self.call("getEncryptData", data=data, key=key)["result"]

    def close(self):
        self.proc.terminate()
```

## 参数拼接规则

JCAP 的 `tk` 和 `ct` 不是简单 JSON，而是**固定格式拼接**后 WASM 加密。

### ct (设备指纹)

```python
# ct = encrypt(random_prefix + sessionId_with_length + DEV_fingerprint + timestamp)
ct_raw = cr(ts % 19) + cm(len(si), 4) + si + DEV + str(ts)
ct = w.encrypt(ct_raw, TDAT_CTX)
```

### tk (轨迹数据)

```python
# tk = encrypt(timestamp + sessionId + state_token + track_data + touch_events + random_suffix)
tk_raw = (str(ts)
          + cm(len(si), 4) + si
          + cm(len(st), 4) + st
          + cm(len(te), 6) + te
          + tj
          + cr(ts % 41))
tk = w.encrypt(tk_raw, TDAT_CTX)
```

### 辅助函数

```python
CHARS35 = "0123456789abcdefghijklmnopqrstuvwxyz"[:35]
cr = lambda n: "".join(random.choice(CHARS35) for _ in range(n))  # random string
cm = lambda v, w: str(v).zfill(w)                                  # zero-padded
eu = lambda s: urllib.parse.quote(s, safe=";,/?:@&=+$-_.!~*'()#")  # URL encode
```

### DEV 设备指纹

```python
DEV = json.dumps({
    "capfp": {}, "cvs": "<canvas_hash>", "wgl": "<webgl_hash>",
    "pr": "1", "cd": "24", "fv": "", "fts": "<font_list>",
    "scr": "1920x1080,1920x1080", "cpu": "8", "pt": "Win32",
    "tzo": "Asia/Shanghai", "lan": "zh-CN",
    "wvr": "Google Inc. (NVIDIA)~ANGLE (NVIDIA, ...)",
    "wdr": False, "mem": 8, "sdv": "2.0", "lns": "zh-CN", "tsp": "0",
}, separators=(",", ":"))
```

需要从真实浏览器抓包获取 `cvs`(canvas hash)、`wgl`(WebGL hash)、`fts`(字体列表)、`wvr`(WebGL renderer)。

### TDAT_CTX

WASM 加密用的密钥上下文，从浏览器 Network 抓包中的 `getEncryptData` 调用参数提取，或从 JS 源码中搜索。格式为长十六进制字符串。

## 接口详解

### 1. getSessionId

```
POST /jnos.account.captcha.getSessionId   (登录场景)
POST /jnos.account.outer.captcha.getSessionId  (业务场景，需签名)
Content-Type: application/json
Body: {"appId": "<captcha_app_id>"}
Response: {"code": 0, "data": {"sessionId": "..."}}
```

不同场景的 `appId` 不同：
- 登录验证码：纯数字 appId（如 `"435228571"`）
- 业务验证码：可能需要额外的 sign 参数

### 2. fp (指纹上报)

```
POST https://captcha-api-global.jdcloud.com/cgi-bin/api/fp
Content-Type: application/x-www-form-urlencoded;charset=UTF-8
x-jdcloud-captcha-auth: <w.auth(app_id)>

Body: si=<sessionId>&ct=<encrypted_ct>&version=2&lang=1&client=m
Response: {"code": 0, "tp": 9, "st": "<state_token>"}
```

- `tp=9`: 无感验证模式（需继续到 check 才能获得 vt）
- `st`: 状态 token，后续 check 请求需要

### 3. check (验证)

```
POST https://captcha-api-global.jdcloud.com/cgi-bin/api/check
Content-Type: application/x-www-form-urlencoded;charset=UTF-8
x-jdcloud-captcha-auth: <w.auth(app_id)>

Body: si=<sessionId>&lang=1&tk=<encrypted_tk>&ct=<encrypted_ct>&version=2&client=m
```

响应类型：
- `tp=9, vt=<token>`: 无感直接通过（理想情况）
- `tp=4, img=<json>`: 降级到滑块验证，img 含 b1(背景) b2(滑块) base64
- `tp=4, vt=<token>`: 滑块验证通过
- `code=16801`: session 过期

## 滑块图片与缺口检测

```python
img = json.loads(d3a["img"])  # img 字段是 JSON 字符串
bg = base64.b64decode(img["b1"].split(",")[-1] if "," in img["b1"] else img["b1"])
piece = base64.b64decode(img["b2"].split(",")[-1] if "," in img["b2"] else img["b2"])

import ddddocr
det = ddddocr.DdddOcr(det=False, ocr=False, show_ad=False)
r = det.slide_match(piece, bg, simple_target=True)
gap_x = r.get("target", [0])[0]  # 原图坐标
```

### 坐标转换

图片原始宽度（如 275px）与显示宽度（如 421px）不同，需要缩放：

```python
DISPLAY_W = 421   # 显示宽度 (从页面 CSS 获取)
ORIG_W = 275      # 原图宽度 (从图片实际尺寸获取)
SCALE = DISPLAY_W / ORIG_W
gap_disp = int(round(gap_x * SCALE))
```

### 偏移修正

ddddocr 检测结果通常有偏差，使用多次尝试 + 偏移：

```python
offsets = [-30, 0, -20, -10, 10, 20, -40, 30]
for offset in offsets:
    test_x = gap_disp + offset
    # submit check with test_x as target
    if vt:  # got verify token
        break
    # if new image returned (tp=4 + img), re-detect gap
```

## 轨迹生成

```python
def gen_track(target_x, steps=25):
    target_x = int(round(target_x))
    track = [[0, 0, 0]]  # [x, y, dt_ms]
    for i in range(1, steps):
        p = i / (steps - 1)
        ease = p * (2 - p)  # ease-out quadratic
        x = int(round(target_x * ease))
        y = random.randint(-1, 1)
        dt = random.randint(22, 35)
        track.append([x, y, dt])
    track[-1] = [target_x, 0, random.randint(25, 40)]
    return track
```

轨迹格式：
```python
track_obj = {
    "ht": 260, "wt": 421,    # 显示高宽
    "bw": 78,  "sw": 421,    # 滑块宽、滑道宽
    "mw": 78,                 # 最大宽度
    "list": gen_track(target_x)
}
te = eu(json.dumps(track_obj, separators=(",", ":")))
```

### touchList (点击事件)

```python
tj = json.dumps({"touchList": [{
    "eid": "click", "did": "", "cn": "send-code",
    "sx": random.randint(1300, 1500), "sy": random.randint(600, 700),
    "px": random.randint(1300, 1500), "py": random.randint(500, 600),
    "time": int(time.time() * 1000) - random.randint(3000, 8000)
}]}, separators=(",", ":"))
```

## 时间戳处理

JCAP 要求时间戳截断到秒（去掉毫秒部分）：

```python
ts = int(time.time() * 1000)
ts = ts - (ts % 1000)  # 截断毫秒
```

## 业务集成示例

### 验证 token 用于登录

```python
# vt 用于 SMS 发送
sms_body = {
    "mobile": encrypted_mobile,
    "scene": 2,
    "slideVerifyCode": vt,      # verify token
    "sessionId": session_id,
    "appId": app_id,
}
```

### 验证 token 用于业务操作

```python
# vt 用于领券等业务
headers = {
    "jnos-verify-app-id": str(captcha_app_id),
    "jnos-verify-id": session_id,
    "jnos-verify-token": vt,
}
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| WASM bridge 崩溃 | jcap.js 版本不兼容 / Node.js 环境缺失 | 重新从浏览器抓取最新 jcap.js |
| fp 返回 code!=0 | DEV 指纹异常 | 从真实浏览器抓包更新 DEV 字段 |
| 一直 tp=9 不出滑块 | 风控等级低，无需滑块 | 直接在 check 响应中拿 vt |
| check 返回 16801 | sessionId 过期 | 重新获取 sessionId |
| 滑块验证一直失败 | 缺口检测偏差 / 坐标缩放错误 | 检查 SCALE 比例、调整 offsets |
| 新图片循环出现 | 检测到自动化特征 | 轨迹加随机性、增加间隔时间 |

## 与其他验证码对比

| 特性 | JCAP (京东云) | TCaptcha (腾讯) | DingXiang (顶象) |
|------|--------------|----------------|-----------------|
| 加密方式 | WASM (必须桥接) | TDC.js (可扣代码) | greenseer.js (Node VM) |
| 指纹上报 | fp 接口 | 内置 TDC | ac 参数 |
| 图片格式 | base64 JSON | URL 下载 | 乱序切割 |
| 缺口检测 | ddddocr | YOLO | OpenCV 模板匹配 |
| 难点 | WASM 加密不可逆 | PoW 计算 | 图片还原算法 |

## 依赖

- Python: `requests`, `ddddocr`, `pycryptodome`(如有 RSA 加密)
- Node.js: 无额外依赖（jcap.js 自包含 WASM）
- 文件: `jcap.js`(从目标站点获取), `crypto_bridge.js`(桥接脚本)

Last updated: 2026-03-20
