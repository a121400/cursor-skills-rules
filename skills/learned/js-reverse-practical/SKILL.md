---
name: js-reverse-practical
description: JS 逆向实战教程。扣代码流程、加密参数定位、Python 集成、RPC 通信、JS Recon 工具、反混淆 AST 还原。来源：腾讯云实战教程 + JS Recon 文档。
---

# JS 逆向实战教程

## 核心流程：从定位到复现

```
1. Network 抓包 → 找加密参数 (token/sign/pwd/nonce)
2. Initiator 回溯 → 定位加密函数入口
3. Sources 搜索 → Ctrl+Shift+F 搜参数名
4. 断点调试 → 单步跟进加密逻辑
5. 扣代码 → 提取完整加密函数 + 依赖
6. 本地验证 → Node.js / Python execjs 运行
7. 封装接口 → 集成到爬虫/自动化脚本
```

## 寻找加密入口

### 优先：jshook 请求回溯（有 MCP 时）

1. `network_get_requests` 列出请求，找到带加密参数的请求。
2. `network_get_request_initiator(requestId)` 直接拿该请求的 JS 调用栈，无需断点。
3. 调用栈有效则跳到「确认入口」；为空则用下面静态/断点方式。

### 加密参数入口定位（find-crypto-entry 流程）

- **Step 0**：network_get_request_initiator 拿到栈 → 有效则 Step 3。
- **Step 1**：search_in_scripts 搜参数名；结果少则看赋值语句，结果多则搜 interceptors.request.use / XMLHttpRequest.prototype.send / window.fetch。
- **Step 2**：breakpoint_set_on_text({ text: "参数名" }) 或 xhr_breakpoint_set({ url: "api 关键词" })，让用户触发请求后 debugger_wait_for_paused。
- **Step 3**：debugger_get_paused_state、get_call_stack；过滤框架栈（axios/react-dom/vue/jquery），优先看非 node_modules 的 /src、/app。
- **Step 4**：在嫌疑栈帧上 debugger_evaluate 看 arguments，确认有位运算、魔数(0x67452301 等)即加密入口。

- **统一输出格式**：找到入口后按下面格式输出：
  - 参数：（参数名）
  - 脚本：（URL）
  - 位置：第 X 行，第 Y 列（或行号+列号）
  - 函数：anonymous 或函数名
  - 调用路径：request → addSign → encrypt
- **说明**：当用户问「xxx 参数在哪生成」时，将「xxx」作为参数名代入上述流程中的搜索/断点步骤；本 skill 只负责定位入口，不做算法还原。

### 方法一：Network 请求分析（无 jshook）

1. 打开 DevTools → Network → 勾选 XHR/Fetch
2. 触发目标操作 (登录/搜索/提交)
3. 找到目标请求，查看 Payload 中的加密参数
4. 点击 Initiator 标签，回溯调用栈到加密函数

### 方法二：全局搜索

- `Ctrl+Shift+F` 搜参数名: `"sign"`, `"token"`, `"pwd"`, `"encrypt"`
- 搜加密特征: `"CryptoJS"`, `"JSEncrypt"`, `"md5"`, `"btoa"`, `"sm2"`

### 方法三：断点拦截

```
XHR 断点:  Sources → XHR/fetch Breakpoints → 添加 URL 关键词
事件断点:  Sources → Event Listener Breakpoints → Mouse → click
条件断点:  右键行号 → Add conditional breakpoint → 输入条件表达式
```

## 扣代码技巧

### 原则

1. 加密函数 + 所有依赖函数必须完整提取
2. 全局变量/常量一并提取 (如 `chrsz = 8`)
3. 如有 Webpack 模块依赖，需要构建简易 `require` 环境
4. 补全浏览器环境 (`window`, `navigator`, `document` 等缺失对象)

### 简单加密：直接扣函数

```javascript
// 示例：MD5 加密扣取
// 1. 在 Sources 中找到 hex_md5 函数
// 2. 右键 → Save as... 或手动复制
// 3. 确保 core_md5, str2binl, binl2hex, chrsz 都被提取

function hex_md5(s) {
    return binl2hex(core_md5(str2binl(s), s.length * chrsz));
}
// ... 所有依赖函数
var chrsz = 8;
```

### Webpack 打包代码：构建加载器

```javascript
// 1. 找到 webpackJsonp / webpackChunk 入口
// 2. 提取目标模块 ID 对应的函数
// 3. 构建简易加载器

var _modules = { /* 粘贴需要的模块 */ };
function _require(id) {
    var mod = { exports: {} };
    _modules[id].call(mod.exports, mod, mod.exports, _require);
    return mod.exports;
}
var encrypt = _require("target_module_id");
```

### 环境补全模板

```javascript
// Node.js 环境补全 (扣代码运行时)
var window = global;
var navigator = { userAgent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" };
var document = { cookie: "", createElement: function() { return { style: {} }; } };
var location = { href: "https://target.com", protocol: "https:" };
```

## Python 集成

### 方案一：execjs 调用 JS (简单加密)

```python
import execjs

with open('extracted_encrypt.js', 'r', encoding='utf-8') as f:
    js_code = f.read()

ctx = execjs.compile(js_code)
result = ctx.call('encryptFunction', plaintext)
```

### 方案二：subprocess 调 Node.js (复杂场景)

```python
import subprocess, json

def call_node(func_name, *args):
    script = f"const m = require('./encrypt.js'); console.log(JSON.stringify(m.{func_name}(...{json.dumps(args)})))"
    result = subprocess.run(['node', '-e', script], capture_output=True, text=True)
    return json.loads(result.stdout.strip())
```

### 方案三：纯 Python 复现 (标准算法)

```python
import hashlib
# MD5
hashlib.md5("password".encode()).hexdigest()

# SHA256
hashlib.sha256("data".encode()).hexdigest()

# AES (pycryptodome)
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
cipher = AES.new(key, AES.MODE_CBC, iv)
ciphertext = cipher.encrypt(pad(plaintext, AES.block_size))

# RSA
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
key = RSA.import_key(public_key_pem)
cipher = PKCS1_v1_5.new(key)
ciphertext = cipher.encrypt(plaintext)

# SM2/SM3/SM4 (gmssl)
from gmssl import sm2, sm3, sm4
```

### 方案四：RPC 桥接 (Sekiro / websocket)

浏览器端注入：
```javascript
// Tampermonkey 脚本
var ws = new WebSocket("ws://localhost:5678");
ws.onmessage = function(e) {
    var req = JSON.parse(e.data);
    var result = window.targetEncrypt(req.params);
    ws.send(JSON.stringify({ id: req.id, result: result }));
};
```

Python 端调用：
```python
import websocket, json, uuid

ws = websocket.create_connection("ws://localhost:5678")
req_id = str(uuid.uuid4())
ws.send(json.dumps({"id": req_id, "params": "plaintext"}))
result = json.loads(ws.recv())
```

## 常见加密类型速查

| 特征 | 算法 | 复现方式 |
|------|------|----------|
| 32位hex | MD5 | hashlib.md5 |
| 40位hex | SHA1 | hashlib.sha1 |
| 64位hex | SHA256 | hashlib.sha256 |
| Base64 结尾有 `=` | Base64 | base64.b64encode |
| 固定长度 + IV 参数 | AES | pycryptodome AES |
| 公钥加密，输出很长 | RSA | pycryptodome RSA |
| `doEncrypt`/`doDecrypt` | SM2 | gmssl.sm2 |
| 64位hex + `invalid mode` | SM3 | gmssl.sm3 |
| `sm4` 关键词 + CBC/ECB | SM4 | gmssl.sm4 |
| `sign` 参数拼接排序 | HMAC/自定义签名 | 扣 JS 逻辑复现 |

## 反混淆工具与方法

### 在线工具

- **deobfuscate.io**: 自动识别 obfuscator.io 混淆并还原
- **deobfuscate.relative.im**: 通用反混淆
- **AST Explorer** (astexplorer.net): 分析 JS 语法树
- **beautifier.io**: 代码格式化 (处理压缩代码)
- **CyberChef**: 多功能编解码工具

### AST 还原 (Babel)

```javascript
// 用 Babel 遍历 AST 还原混淆
const { parse } = require("@babel/parser");
const { default: traverse } = require("@babel/traverse");
const { default: generate } = require("@babel/generator");

const ast = parse(obfuscatedCode);

// 还原字符串加密
traverse(ast, {
    CallExpression(path) {
        // 识别解密函数调用，替换为明文字符串
        if (isDecryptorCall(path.node)) {
            const result = evaluateDecryptor(path.node);
            path.replaceWith({ type: "StringLiteral", value: result });
        }
    }
});

// 还原控制流平坦化
traverse(ast, {
    WhileStatement(path) {
        // 识别 switch-case 状态机，按执行顺序重排代码块
    }
});

const { code } = generate(ast);
```

### 混淆类型与算法特征（jshook 可配合 detect_obfuscation/detect_crypto）

| 混淆特征 | 类型 |
|----------|------|
| _0x/a0_ 前缀、大字符串数组 + 索引访问、while(true)+switch、多层 eval、jsjiami.com、WASM、自定义字节码 | OB/sojson/控制流平坦化/VM/WASM |
| 32/40/64 位 hex、0x67452301、S 盒 0x63,0x7c、ipad/opad、RSA 关键词 | MD5/SHA1/SHA256/AES/HMAC/RSA |

### JS 转 Python 注意

- charCodeAt(i)→ord(s[i])，fromCharCode(n)→chr(n)，>>> 无符号右移→(x & 0xFFFFFFFF) >> n。
- 32 位：`def to_int32(x): x=x&0xFFFFFFFF; return x if x<0x80000000 else x-0x100000000`

### 常见混淆模式

| 混淆类型 | 识别特征 | 还原方法 |
|----------|----------|----------|
| 变量名混淆 | `_0x1234`, `$_IJDD` | beautifier 格式化后手动重命名 |
| 字符串加密 | 大量函数调用替代字符串 | AST 遍历执行解密函数（详见 ast-deobfuscate） |
| 控制流平坦化 | while-switch-case 嵌套 | AST 按状态顺序重排 |
| 死代码注入 | 永远不执行的分支 | AST 删除不可达代码 |
| 数组旋转 | 字符串存数组 + 移位函数 | 执行移位后内联替换 |

## JS Recon 工具 (v1.2.1)

JS Recon 是专门用于 JavaScript 侦察的工具，从 JS 文件中提取有价值信息。

### 核心命令

| 命令 | 功能 |
|------|------|
| `run` | 自动运行所有核心模块 |
| `map` | 映射和分析 JS 文件中的函数 |
| `strings` | 提取字符串、URL、密钥 |
| `endpoints` | 提取客户端端点 |
| `lazyload` | 下载所有懒加载的 JS 文件 |
| `api-gateway` | 配置 AWS API Gateway 做 IP 轮转 |

### 典型用法

```bash
# 自动分析目标网站所有 JS
js-recon run -u https://target.com

# 提取 JS 中的 API 端点
js-recon endpoints -u https://target.com

# 提取密钥和敏感字符串
js-recon strings -u https://target.com

# 分析函数结构 (Next.js 支持交互模式)
js-recon map -u https://target.com
```

### 适用场景

- 快速发现隐藏 API 端点
- 提取硬编码密钥/token
- 分析 JS 文件结构，定位加密函数
- 下载 SPA 应用的所有 JS 资源

## 完整实战检查清单

```
[ ] 1. Network 抓包，找到目标请求和加密参数
[ ] 2. Initiator 回溯 / 全局搜索定位加密入口
[ ] 3. 开 AntiDebug Breaker (如有反调试)
[ ] 4. 开加密 Hook (CryptoJS/JSEncrypt/SM)
[ ] 5. 断点调试，单步跟进加密逻辑
[ ] 6. 判断加密类型 (标准算法 / 自定义)
[ ] 7. 标准算法 → Python 库直接复现
[ ] 8. 自定义算法 → 扣 JS 代码 + 环境补全
[ ] 9. 本地运行验证加密结果一致
[ ] 10. 集成到爬虫/自动化脚本
[ ] 11. 复杂场景备选: RPC 桥接 / headed Chrome
```

Last updated: 2026-02-27
