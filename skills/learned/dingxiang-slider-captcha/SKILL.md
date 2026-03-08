---
name: dingxiang-slider-captcha
description: 顶象 (DingXiang) 滑块验证码逆向与绕过。包含完整流程：获取验证码挑战、图片乱序还原、OpenCV 缺口识别、greenseer.js ac 参数生成、Node.js VM 沙箱执行、轨迹模拟。适用于美的/小天鹅等使用顶象验证码的站点。
---

# 顶象滑块验证码逆向 (实战总结)

## 识别特征

- API 域名: `sec.midea.com` 或目标站 `/api/a` + `/api/v1`
- 参数: `ak` (appKey), `c` (constId), `jsv`, `aid`, `sid`
- 背景图乱序切割，需还原
- ac 参数由 `greenseer.js` + `_dx.UA` 生成

## 完整流程

```
1. agree_agreement()    获取 checkId (协议同意接口)
2. get_captcha()        获取验证码挑战 (sid/p1/p2/y)
3. download + 还原图片   下载背景图和滑块图，还原乱序背景
4. opencv 识别缺口       OpenCV 模板匹配获取 x 坐标
5. get_ac()             Node.js VM 沙箱执行 greenseer.js 生成 ac
6. slide_verify()       提交 ac/sid/x/y 获取 token
7. send_sms()           携带 token 发送短信
```

## 第1步: 获取验证码挑战

```python
url = f"{BASE_URL}/api/a?w=366&h=150&s=50&ak={APP_KEY}&c={CONST_ID}&jsv={JSV}&aid={aid}&wp=1&de=0&lf=0&t={T_PARAM}&cid=04183840&_r={random.random()}"
```

返回:
```json
{
  "p1": "/xxx.webp",     // 背景图路径 (乱序)
  "p2": "/xxx.png",      // 滑块图路径
  "sid": "xxx",          // 会话 ID
  "y": 50                // 滑块 y 坐标
}
```

图片下载: `{BASE_URL}/picture{p1}`, `{BASE_URL}/picture{p2}`

## 第2步: 背景图乱序还原

顶象将 380px 宽背景图切成 32 块 (每块 12px)，按文件名计算乱序:

```python
def _calc_order(name_str):
    """根据图片文件名字符计算切割顺序"""
    result = []
    for i in range(len(name_str)):
        c = ord(name_str[i])
        if i == 32:
            break
        while c % 32 in result:
            c += 1
        result.append(c % 32)
    return result

def pic_re(pic_path, bg_url):
    """还原乱序背景图"""
    img = Image.open(pic_path)
    new_img = Image.new('RGB', (380, 165))
    name = bg_url.split("/")[-1].split(".")[0]
    order = _calc_order(name)
    for idx, src_idx in enumerate(order):
        x = src_idx * 12
        img_cut = img.crop((x, 0, x + 12, 200))
        new_img.paste(img_cut, (idx * 12, 0))
    new_img.save('./img/re_bg.jpg')
```

## 第3步: OpenCV 缺口识别

```python
import cv2
import numpy as np

def get_distance(bg_bytes, tp_bytes):
    """Canny 边缘检测 + 模板匹配"""
    bg_img = cv2.imdecode(np.frombuffer(bg_bytes, np.uint8), 1)
    tp_gray = cv2.cvtColor(
        cv2.imdecode(np.frombuffer(tp_bytes, np.uint8), 1),
        cv2.COLOR_BGR2GRAY
    )
    bg_shift = cv2.pyrMeanShiftFiltering(bg_img, 5, 50)
    tp_edge = cv2.Canny(tp_gray, 255, 255)
    bg_edge = cv2.Canny(bg_shift, 255, 255)
    result = cv2.matchTemplate(bg_edge, tp_edge, cv2.TM_CCOEFF_NORMED)
    _, _, _, max_loc = cv2.minMaxLoc(result)
    return max_loc[0]

# 实际使用时需微调偏移: distance = opencv2() - 14
```

## 第4步: ac 参数生成 (核心)

### 架构

```
browser_env.js     浏览器环境模拟 (window/navigator/document/DOM)
env.js             顶象 SDK 环境补全
greenseer.js       顶象加密核心 (动态下载: /ctu-group/ctu-greenseer/greenseer.js)
code_template.js   轨迹生成 + _dx.UA 调用模板
runner.js          Node.js VM 沙箱执行器
```

### 轨迹生成

```javascript
function slide_track(distance) {
    // ease_out_expo 缓动函数模拟人手滑动
    var track = [[372, 507, 3526]];  // 起始点 [x, y, timestamp]
    var count = 30 + parseInt(distance / 2);
    var t = random_randint(50, 100);
    for (var i = 0; i < count; i++) {
        var x = Math.round(ease_out_expo(i / count) * distance);
        t += random_randint(30, 50);
        track.push([372 + x, 507, 3526 + t]);
    }
    return track;
}
```

### _dx.UA 调用链

```javascript
function get_result() {
    var mousemove = slide_track(pic_x);
    // 1. 初始化 UA 实例
    var ua = window['_dx'].UA.init({ "token": sid });
    // 2. 逐点记录鼠标轨迹
    for (var i = 0; i < mousemove.length; i++) {
        ua.__proto__.tm = new Date().getTime() - mousemove[i][2];
        ua.recordSA({pageX: mousemove[i][0], pageY: mousemove[i][1]});
    }
    ua.sendSA();
    // 3. 发送最终坐标
    ua.sendTemp({ xpath: '/html/body/div[11]', x: pic_x, y: pic_y });
    // 4. 获取 ac 参数
    return ua.getUA();
}
```

### Node.js VM 沙箱执行

```javascript
// runner.js 关键设计
var sandbox = {
    console: silentConsole,  // 静默 console 避免污染 stdout
    Buffer, Date, Math, JSON, Array, Object, String, Number,
    Promise, Proxy, Reflect, Symbol, Map, Set, ...
    // 不含 process/require/__dirname 避免 Node.js 检测
};
sandbox.global = sandbox;   // 自引用模拟浏览器 global
sandbox.sid = args[0];
sandbox.pic_x = parseInt(args[1]);
sandbox.pic_y = parseInt(args[2]);

vm.createContext(sandbox);
// 按顺序执行: browser_env.js -> env.js -> _code_runtime.js
// 最后调用 get_result() 获取 ac
```

### greenseer.js 动态注入

```python
# Python 端: 下载最新 greenseer.js 并注入到模板
gs_url = f"{BASE_URL}/ctu-group/ctu-greenseer/greenseer.js?_t=486973"
greenseer = session.get(gs_url).text

# 注入到 code_template.js 的 /* start */ ... /* end */ 之间
code = template[:start_idx] + '/* start */\n' + greenseer + '\n/* end */' + template[end_idx:]
```

## 第5步: 提交验证

```python
data = {
    'ac': ac,           # get_result() 返回值
    'ak': APP_KEY,
    'c': CONST_ID,
    'uid': '',
    'jsv': JSV,
    'sid': sid,
    'aid': aid,
    'x': str(x),
    'y': str(y),
}
resp = session.post(f"{BASE_URL}/api/v1", data=data)
# 成功: {"success": true, "token": "xxx"}
# 失败: {"success": false, "msg": "xxx"}
```

成功后用 `token:constId` 拼成 `sliderCode` 传给业务接口。

## 环境补全清单

browser_env.js 中必须模拟的对象:

| 对象 | 关键属性 |
|------|----------|
| window | top/self/addEventListener/getComputedStyle/innerWidth/innerHeight |
| navigator | userAgent/platform/webdriver=false/languages/plugins/hardwareConcurrency |
| document | createElement/getElementById/querySelector/cookie/body/documentElement |
| screen | width=1920/height=1080/availWidth/availHeight/colorDepth=24 |
| location | href/origin/protocol/host/hostname |
| DOM 元素 | offsetWidth/offsetHeight/getBoundingClientRect/style/classList |
| canvas | getContext('2d')/toDataURL (返回固定 base64) |
| performance | now()/timing |

## 常见问题

| 问题 | 解决 |
|------|------|
| ac 参数为空 | 检查 greenseer.js 是否正确注入，browser_env.js 是否完整 |
| 滑块验证失败 | 微调缺口偏移量 (distance - 14)，检查轨迹点数是否合理 |
| Node.js 执行报错 | 检查 env.js 中 documentall 是否定义 |
| greenseer.js 变更 | 每次动态下载最新版本，不要硬编码 |
| token 过期 | 整个流程需在短时间内完成 (建议 30s 内) |

## 依赖

```
Python: requests, Pillow, opencv-python, numpy, loguru
Node.js: 内置 vm/fs/path 模块 (无需额外安装)
```

Last updated: 2026-03-07
