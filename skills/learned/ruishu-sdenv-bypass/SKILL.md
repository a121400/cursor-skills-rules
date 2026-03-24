---
name: ruishu-sdenv-bypass
description: "瑞数 (Ruishu) WAF Cookie + T参数生成方案。使用 sdenv 补环境框架在 Node.js 中生成有效的瑞数 cookie 和动态 URL 参数(jhsgvYT0)，绕过瑞数 VMP 反爬。支持微信浏览器等自定义环境模拟。当逆向分析发现目标站点使用瑞数 WAF（特征：$_ts 变量、yrXXX 格式 cookie、412 状态码拦截、动态 JS VM 代码、URL 中带 jhsgvYT0 等动态参数）时使用。"
---

# 瑞数 WAF Cookie + T参数生成 (sdenv 补环境方案)

## 识别瑞数 WAF

特征：
- 首次请求返回挑战页面，HTML 中包含 `$_ts` 全局变量（含 `cd`、`nsd` 等字段）
- cookie 名形如 `yrXXXXXXXXXXO` 和 `yrXXXXXXXXXXP`（末尾 O/P/T）
- 无有效 cookie 时返回 HTTP 412
- 页面加载动态 JS（含 VMP 虚拟机代码），执行后通过 `location.replace` 跳转
- API 请求 URL 带动态参数（如 `jhsgvYT0=...`），称为 **T参数**

## 方案选择

| 方案 | 适用性 | 说明 |
|------|--------|------|
| **sdenv 补环境** | 通杀，推荐 | jsdom 模拟浏览器，瑞数 JS 自己跑出 cookie + T参数 |
| v_jstools / 手动补环境 | 不推荐 | VMP 字节码解释器对环境极敏感，`RangeError: Invalid array length` |
| rs-reverse 纯算法 | 仅部分版本 | 需要适配 basearr，不同站点差异大，维护成本高 |
| Playwright 浏览器 RPC | 备选 | 真实浏览器获取，但需要浏览器实例，资源消耗大 |

**sdenv 通杀瑞数**（4/5/6 代均可），纯 Node.js 运行，约 7 秒生成。

## 核心原则：环境必须与真实浏览器一致

sdenv 默认模拟 Mac Chrome，但**实际访问环境**可能不同。环境与 UA 不一致会被 VMP 指纹检测拦截。

**决策流程：**
1. 确认目标站点的真实访问方式（普通 Chrome / 微信浏览器 / APP WebView / 小程序）
2. UA 中含 `MicroMessenger` / `WindowsWechat` -> 必须配置微信环境（见下方 beforeParse 节）
3. 普通 Chrome 访问 -> sdenv 默认环境即可，无需额外配置
4. 所有 `jsdomFromUrl` 调用都要传相同的 `beforeParse`，保持环境一致

## 环境要求

- **Node.js >= v20.19.5**（ESM 支持，低版本会报 ERR_REQUIRE_ESM）
- Python（node-gyp 编译需要）
- Visual Studio + "使用 C++ 的桌面开发"（Windows native addon 编译）

当前可用 Node 路径：`C:\node20\node-v20.19.5-win-x64`

## 安装

```bash
npm init -y
npm install sdenv
```

安装时 canvas 可能从 GitHub 下载预编译包较慢，设置代理或换源。canvas 安装失败不会中断，但运行时调用 canvas API 会报错。

## 核心用法：生成 Cookie

```javascript
process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0";
const { jsdomFromUrl } = require('sdenv');

const tarUrl = "https://target-site.com/path";
const userAgent = 'Mozilla/5.0 ...'; // 与业务请求保持一致

async function getCookie(url) {
  return new Promise(async (resolve, reject) => {
    const timeout = setTimeout(() => reject(new Error('30s timeout')), 30000);
    const { window, cookieJar } = await jsdomFromUrl(url, { userAgent });

    window.addEventListener('sdenv:exit', (e) => {
      if (['location.replace', 'location.assign'].includes(e.detail.eventId)) {
        clearTimeout(timeout);
        const cookies = cookieJar.getCookieStringSync(url);
        window.close();
        resolve(cookies);
      }
    });
  });
}
```

### 关键点

1. `jsdomFromUrl` 请求目标 URL，sdenv 自动补全浏览器环境
2. 瑞数 JS 在 jsdom 中执行，生成 cookie 后触发 `location.replace`
3. 监听 `sdenv:exit` 事件获取 cookie
4. `cookieJar.getCookieStringSync(url)` 提取最终 cookie 字符串
5. **User-Agent 必须与后续请求一致**，cookie 中编码了 UA 信息

## 自定义浏览器环境 (beforeParse)

**关键：sdenv 默认 = Mac Chrome。微信站点必须用 beforeParse 补齐微信环境，否则 UA 说微信但环境像 Chrome，VMP 会检测到不一致。**

### beforeParse 机制

`jsdomFromUrl` 支持 `beforeParse(window, sdenv)` 回调，在 sdenv 浏览器环境注入**之后**、VMP 代码执行**之前**调用：

```javascript
const { jsdomFromUrl } = require('sdenv');
const dom = await jsdomFromUrl(url, {
  userAgent: '...',
  beforeParse(window, sdenv) {
    // sdenv 浏览器环境已注入，可以覆盖/追加属性
    // sdenv.tools.setFuncNative() 可伪装函数为 native
  },
});
```

### navigator 属性覆盖注意事项

sdenv 的 `mixin` 把 navigator 属性定义为 `window.navigator` 的**自有属性**（own property），
必须在 `nav` 对象本身上重新 defineProperty 才能覆盖，仅在原型链上定义无效：

```javascript
beforeParse(window, sdenv) {
  const nav = window.navigator;
  const proto = Object.getPrototypeOf(nav);
  const desc = { get: () => 'Win32', configurable: true };
  // 必须两层都定义，自有属性优先
  Object.defineProperty(nav, 'platform', desc);
  Object.defineProperty(proto, 'platform', desc);
}
```

### 微信浏览器环境补丁

微信内置浏览器（Windows WeChat / XWEB）的关键特征：

| 属性 | 值 |
|------|----|
| `navigator.platform` | `Win32` (非 MacIntel) |
| `navigator.userAgent` | 含 `MicroMessenger`、`WindowsWechat`、`XWEB` |
| `WeixinJSBridge` | 微信 JS Bridge 对象 (invoke/call/on/log) |
| `wx` | 微信 JS-SDK (config/ready/miniProgram) |
| `__wxjs_is_wkwebview` | `true` |
| `screen` | 1920x1080 (Windows 桌面) |
| `innerWidth/Height` | 414x736 (微信窗口) |

使用方式（每个 `jsdomFromUrl` 调用都要加）：

```javascript
const { createWechatBeforeParse } = require('./wechat_env');
const { window, cookieJar } = await jsdomFromUrl(url, {
  userAgent: UA,
  beforeParse: createWechatBeforeParse(UA),
});
```

完整补丁实现（`wechat_env.js`）：

```javascript
function createWechatBeforeParse(ua) {
  return function(window, sdenv) {
    const nav = window.navigator;
    const proto = Object.getPrototypeOf(nav);
    // 覆盖 navigator (nav + proto 两层)
    for (const [key, getter] of Object.entries({
      userAgent: () => ua,
      appVersion: () => ua.replace('Mozilla/', ''),
      platform: () => 'Win32',
    })) {
      const desc = { get: getter, configurable: true };
      try { Object.defineProperty(nav, key, desc); } catch(_) {}
      try { Object.defineProperty(proto, key, desc); } catch(_) {}
    }
    // WeixinJSBridge
    window.WeixinJSBridge = { invoke(){}, call(){}, on(){}, log(){} };
    // wx JS-SDK
    window.wx = {
      config(){}, ready(cb){ cb&&cb(); }, error(){},
      miniProgram: { getEnv(cb){ cb&&cb({miniprogram:false}); } },
    };
    window.__wxjs_is_wkwebview = true;
  };
}
```

### windowProxyConfig

sdenv 的 window 代理可隐藏 Node.js 特征，默认已配置 `process` 抛 ReferenceError：

```javascript
// 默认值（globalVarible.js 中）：
windowGetterErrorKeys: ['process'],
windowGetterUndefinedKeys: ['_runScripts', '_globalObject', ...jsdom内部属性],
windowGetterWinKeys: ['window', 'top', 'self', 'frames', 'globalThis'],
```

如需额外配置，通过 `windowProxyConfig` 传入：

```javascript
await jsdomFromUrl(url, {
  userAgent: ua,
  beforeParse: myPatch,
  windowProxyConfig: {
    windowGetterErrorKeys: ['process', 'Buffer'],
    windowGetterUndefinedKeys: ['_runScripts'],
  },
});
```

## T参数生成 (jhsgvYT0)

部分瑞数站点的 API 请求需要动态 URL 参数（如 `jhsgvYT0`），由 VMP 在 XHR 发送前注入。

### 原理

VMP Hook 了 `XMLHttpRequest.open/send`，在请求发出时自动追加 T参数到 URL。
拦截 `https.request` 可捕获被 VMP 修改后的 URL。

### 实现步骤

1. **第一次加载**：生成 cookie（监听 `sdenv:exit`）
2. **第二次加载**：同 URL + 同 cookieJar 加载，VMP 安装 XHR Hook
3. **拦截 https.request**：捕获 VMP 注入的 T参数，返回模拟响应（不发真实请求）
4. **触发 XHR**：用 `window.XMLHttpRequest` 发请求，VMP 自动注入 T参数

```javascript
// 拦截 https.request，捕获 T参数
const _origReq = https.request;
https.request = function(opts, cb) {
  const path = typeof opts === 'string' ? opts : opts.path || '';
  const match = path.match(/[?&]jhsgvYT0=([^&]*)/);
  if (match) {
    lastTParam = match[1];
    // 返回模拟 200 响应，不发真实请求
    const fakeReq = new PassThrough();
    fakeReq.end = () => {};
    process.nextTick(() => {
      if (cb) {
        const fakeRes = new PassThrough();
        fakeRes.statusCode = 200;
        fakeRes.headers = {'content-type':'application/json'};
        cb(fakeRes);
        fakeRes.end('{"code":0,"data":[]}');
      }
    });
    return fakeReq;
  }
  return _origReq.apply(this, arguments);
};
```

### P cookie 恢复

第二次加载 VMP 会生成较短的新 P cookie（~321 字符），覆盖第一次的有效值（~407 字符）。
需保存第一次的 P cookie，在第二次加载后恢复：

```javascript
const origP = jar.getCookieStringSync(url).match(/yrXXXXP=([^;]+)/)?.[1];
// ... 第二次 jsdomFromUrl ...
// 恢复
if (origP) {
  const tough = require('tough-cookie');
  jar.setCookieSync(new tough.Cookie({
    key: 'yrXXXXP', value: origP, domain: 'target.com', path: '/'
  }), url);
}
```

### Python 集成 (stdin/stdout IPC)

Node.js 作为子进程运行，通过 JSON 行协议与 Python 通信：

```
Python -> Node stdin:  {"path": "/api/xxx", "method": "POST"}
Node -> Python stdout: {"cookies": "...", "tparam": "...", "url": "https://...?jhsgvYT0=..."}
```

控制命令：`{"cmd": "exit"}` 退出，`{"cmd": "reset"}` 重新初始化。

## 验证标准

| HTTP 状态码 | 含义 |
|-------------|------|
| 412 | cookie 无效，被瑞数拦截 |
| 200 | 通过，正常响应 |
| 400 | 通过，业务层报错（参数问题，非瑞数拦截） |

## 排错

| 问题 | 原因 | 解决 |
|------|------|------|
| ERR_REQUIRE_ESM | Node 版本低 | 升级到 v20.19.5+ |
| node-gyp 编译失败 | 缺 C++ 编译环境 | 安装 VS + C++ 桌面开发 |
| content type not HTML/XML | 第二次请求返回 JSON | 改用 https.request |
| 30s 超时无 cookie | 网络问题或站点变化 | 检查网络连通性，确认 $_ts 存在 |
| 412 | UA 不一致或 cookie 过期 | 确保 UA 一致，cookie 有时效性需即时使用 |
| `ReferenceError: process is not defined` | Vue/Webpack 检测 process.env | 正常，sdenv 默认隐藏 process |
| P cookie 变短导致 412 | 第二次 VMP 加载覆盖 P cookie | 保存第一次 P 值，第二次加载后恢复 |
| navigator.platform = MacIntel | sdenv 默认 Mac Chrome 环境 | 用 beforeParse 覆盖为 Win32 |
| `RangeError: Invalid array length` | v_jstools/手动补环境不完整 | 放弃手动补环境，使用 sdenv |
| T参数为空 | VMP XHR Hook 未安装 | 确保第二次加载完成并等待 2s |

## 不推荐的方案

### v_jstools / 手动补环境

通过 Proxy 记录浏览器 API 访问并在 Node.js 中模拟。对瑞数 VMP **不可行**：
VMP 字节码解释器逐字符消费 bytecode 字符串，环境细微差异导致指针偏移，
最终 `RangeError: Invalid array length`。已在 ticket.sxhm.com 上多次验证失败。

## 相关项目

- [pysunday/sdenv](https://github.com/pysunday/sdenv) - 补环境框架
- [pysunday/sdenv-jsdom](https://github.com/pysunday/sdenv-jsdom) - 专用 jsdom（过瑞数检测）
- [pysunday/sdenv-extend](https://github.com/pysunday/sdenv-extend) - 多端环境扩展
- [pysunday/rs-reverse](https://github.com/pysunday/rs-reverse) - 纯算法方案（部分版本）

## 实战参考

已验证站点：`ticket.sxhm.com`（瑞数 6 代，微信浏览器环境）
- cookie 生成：O(88 chars) + P(407 chars)，约 7s
- T参数生成：119 chars，约 8s（含 VMP Hook 加载）
- 微信环境补丁：`beforeParse` 注入 WeixinJSBridge/wx/Win32 平台

Last updated: 2026-03-14
