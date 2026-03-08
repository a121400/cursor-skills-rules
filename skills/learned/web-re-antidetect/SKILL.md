---
name: web-re-antidetect
description: Web 逆向工程与反调试绕过。JS 加密逆向、验证码破解、反 F12 绕过、Cookie/Token 签名分析、浏览器指纹反检测。从 kimi/google/js/lanren/yuabn/Desktop/XC 等 15+ 项目提取。
---

# Web 逆向与反检测 (Auto-Learned)

用户高频任务：分析 Web 端加密/签名算法，绕过反调试，破解验证码，浏览器指纹反检测。

## 典型分析流程

```
1. 用 jshook MCP 连接目标站点
2. 分析反调试机制（F12 禁用、debugger 循环）
3. Hook 关键加密函数（btoa/atob/crypto）
4. 提取签名算法，扣 JS 代码或 Python 复现
5. 处理验证码（滑块/图形/易盾）
```

## 反调试绕过

用户常遇到的反调试手段：
- F12 打开即报错/跳转
- `debugger` 无限循环
- Console 检测

```javascript
// jshook 中常用的反调试绕过
// 1. 禁用 debugger 语句
Function.prototype.constructor = new Proxy(Function.prototype.constructor, {
    apply(target, thisArg, args) {
        if (args[0] && args[0].includes('debugger')) {
            return () => {};
        }
        return Reflect.apply(target, thisArg, args);
    }
});

// 2. 阻止 DevTools 检测
Object.defineProperty(window, 'outerHeight', { get: () => window.innerHeight });
```

## JS 加密逆向模式

| 项目 | 加密类型 | 方法 |
|------|----------|------|
| lotsmall (Desktop) | RSA + 自定义加密 | 扣 JS 代码 + 补环境 |
| kimi (易盾) | 滑块验证 + 签名 | ddocr 识别缺口 + 逆向签名 |
| yuabn | 滑块采集点 | 坐标采集 + 算法校验 |
| google (Gmail) | identity-signin token | 纯协议分析 |
| lanren | validate 处理 | 登录流程 token 链 |
| XC (同程旅行) | User-Dun 指纹签名 | 见下方专项总结 |

## 验证码处理

```python
# 滑块验证码通用模式
import ddddocr

ocr = ddddocr.DdDdOcr(det=False, ocr=False)

def get_slide_distance(bg_bytes, slide_bytes):
    """识别滑块缺口位置"""
    result = ocr.slide_match(slide_bytes, bg_bytes, simple_target=True)
    return result['target'][0]  # x 坐标
```

### 顶象 (DingXiang) 滑块

详见专项技能 `dingxiang-slider-captcha`。核心要点:
- 背景图乱序切割 (32 块 x 12px)，需按文件名算法还原
- OpenCV Canny 边缘 + 模板匹配识别缺口
- ac 参数由 greenseer.js 在 Node.js VM 沙箱中执行生成
- 轨迹用 ease_out_expo 缓动函数模拟人手

## 微信小程序 H5 环境绕过

详见专项技能 `wechat-miniprogram-reverse`。核心要点:
- SunnyNet "响应文件" 规则替换 JS 文件
- SunnyNet "String(UTF8)" 规则改写 HTTP 请求头中的 User-Agent
- JS 层用 Object.defineProperty 覆盖 navigator.userAgent/platform
- Vue created() 中 isMobile() 检测改为 if(!0) 绕过
- 修改要最小化，保留原始业务逻辑避免路由/数据链路断裂

## 高级指纹签名反爬深度总结 (XC/同程 User-Dun 案例)

### 核心教训: 先验证目标接口再投入逆向

**在逆向签名算法之前，必须先确认目标 API 端点在真实浏览器中是否能正常返回数据。**

同程项目暴露的致命问题:
- `tapi/v2/list` 端点: **即使在有头真实 Chrome 中也返回 errorCode -99**，说明该端点本身有额外限制或已废弃
- `tapi/v2/nlist` 端点: 同一页面中**无需 User-Dun 签名**即可正常返回酒店数据
- 页面前端代码会同时请求 list 和 nlist，实际数据从 nlist 获取

**验证步骤 (必须在逆向前做):**
1. 用 puppeteer 拦截页面正常加载时的所有 API 请求和响应
2. 确认哪个端点实际返回了有效数据
3. 确认该端点是否需要签名 header
4. 如果主端点不需要签名，直接走纯协议复现，不必逆向签名算法

### User-Dun 签名算法结构

dun.min.js 的签名生成流程:
1. **指纹采集**: 40+ 浏览器属性 (Canvas/WebGL/Audio/Navigator/Screen/Error.stack 等)
2. **二进制编码**: 自定义 k5 类构建 buffer (D=单字节, C=整数, P=字符串, L=子缓冲区)
3. **加密**: 对 buffer 进行加密
4. **自定义 Base64**: 字符集 `wxyz0123456789abcdefghijklmnopqrstuvABCDEFGHIJKLMNOPQRSTUVWXYZ+/=`
5. 最终输出为 User-Dun header 值

### jsdom 环境补全清单 (即使最终方案不用，也是有价值的参考)

dun.min.js 检测的 Node.js/jsdom 特征:

| 检测项 | Chrome 期望值 | jsdom 默认值 | 修复方式 |
|--------|-------------|-------------|---------|
| `typeof document.all` | `"undefined"` (V8 exotic object) | `"object"` | `Object.defineProperty(doc, 'all', {value: undefined})` |
| `typeof require` | ReferenceError (未声明) | `undefined` 或存在 | `delete win.require` (不能设 undefined) |
| `canvas.toBuffer` | 不存在 | node-canvas 存在 | 删除该方法 |
| `ctx.textDrawingMode` | 不存在 | node-canvas 存在 | 删除该属性 |
| `Error().stack` | Chrome 格式, 无 node: 路径 | 含 node:internal 路径 | `Error.prepareStackTrace` 全局 hook |
| `navigator.webdriver` | `false` | `true` 或 `undefined` | defineProperty get => false |
| `window.chrome` | 存在 (对象) | 不存在 | 创建 mock 对象 |
| `process/global/Buffer/__dirname` | ReferenceError | 存在 | delete 删除 |
| `toDataURL.prototype` | `undefined` (原生函数无 .prototype) | 存在 (polyfill 函数) | `delete HTMLCanvasElement.prototype.toDataURL.prototype` |
| `navigator.maxTouchPoints` | 0 (桌面) 或 10 (触屏) | 0 | 按目标设备设置 |
| `screen.availHeight` | 通常等于 screen.height | 可能不同 | 手动对齐 |

### 页面请求头字段 (易被忽略的必要字段)

同程 H5 页面的 API 请求头 (从浏览器拦截获得):
```
tmapi-client: th5        // 不是 h5
appfrom: 14              // 不是 h5
cluster: idc
timezone: 8
deviceid: <H5CookieId>
pagename: hotellist      // 小写
traceid: <uuid>          // 每次请求唯一
scriptVersion: 2.6.62    // URL 参数
```

### headless 检测实测结果

| 环境 | User-Dun 长度 | list 结果 | nlist 结果 |
|------|-------------|-----------|-----------|
| Chrome headed | 408 | -99 | SUCCESS |
| Chrome headless=new | ~408 | -99 | 未测 |
| jsdom + 补环境 | 676 | -99 | N/A |
| puppeteer headed + 手动 fetch | 可变 | -99 | N/A |

**结论**: list 端点对所有自动化工具返回 -99，包括真实有头 Chrome。nlist 才是有效数据端点。

### 纯协议复现 nlist (推荐方案)

```javascript
// 最小可用方案: Node.js 直连 nlist
const https = require('https');
const crypto = require('crypto');

// 1. 初始化会话
const deviceId = crypto.randomUUID();
// GET https://m.ly.com/hotel/hotellist?city=53 获取 JSESSIONID + route

// 2. 请求 nlist (不需要 User-Dun)
const url = `https://m.ly.com/tapi/v2/nlist?scriptVersion=2.6.62&city=53&inDate=YYYY-MM-DD&outDate=YYYY-MM-DD&pageIndex=0&pageSize=10&sortType=0&isResHotel=1`;
// Headers: tmapi-client=th5, appfrom=14, cluster=idc, timezone=8, deviceid, traceid, pagename=hotellist
// Cookie: H5CookieId + JSESSIONID + route
```

## jshook/Puppeteer 浏览器常见检测点

**必须修复的 6 个检测向量:**

1. **`navigator.permissions.query.toString()` 泄露 stealth 代码**
   - 症状: 返回 stealth 函数源码而非 `[native code]`
   - 修复: Proxy 包裹 + 重写 toString

2. **`navigator.webdriver` 为 undefined 而非 false**
   - 修复: `Object.defineProperty(navigator, 'webdriver', { get: () => false })`

3. **`innerHeight > outerHeight` 窗口尺寸异常**
   - 修复: 确保 outerHeight > innerHeight + 100

4. **`maxTouchPoints` 不匹配设备类型**
   - 桌面应为 0，触屏设备 10

5. **`navigator.languages` 不匹配地区**
   - 中文用户: `["zh-CN", "zh", "en"]`

6. **Plugins 列表过时**
   - Chrome 120+: `["PDF Viewer", "Chrome PDF Viewer", "Chromium PDF Viewer", "Microsoft Edge PDF Viewer", "WebKit built-in PDF"]`

### 推荐的反检测注入脚本

```javascript
// 在 page.evaluateOnNewDocument 中注入 (页面加载前)
(() => {
  Object.defineProperty(navigator, 'webdriver', { get: () => false });

  const origQuery = navigator.permissions.query.bind(navigator.permissions);
  navigator.permissions.query = function(desc) { return origQuery(desc); };
  Object.defineProperty(navigator.permissions.query, 'toString', { 
    value: () => 'function query() { [native code] }',
    writable: false, configurable: false 
  });

  const realInnerH = window.innerHeight;
  Object.defineProperty(window, 'outerHeight', { get: () => realInnerH + 110 });
  Object.defineProperty(window, 'outerWidth', { get: () => window.innerWidth });

  Object.defineProperty(navigator, 'maxTouchPoints', { get: () => 0 });
  Object.defineProperty(navigator, 'languages', { get: () => ['zh-CN', 'zh', 'en'] });
  Object.defineProperty(navigator, 'language', { get: () => 'zh-CN' });

  Object.defineProperty(navigator, 'plugins', {
    get: () => {
      const p = [
        { name:'PDF Viewer', filename:'internal-pdf-viewer', description:'Portable Document Format', length:2 },
        { name:'Chrome PDF Viewer', filename:'internal-pdf-viewer', description:'', length:2 },
        { name:'Chromium PDF Viewer', filename:'internal-pdf-viewer', description:'', length:2 },
        { name:'Microsoft Edge PDF Viewer', filename:'internal-pdf-viewer', description:'', length:2 },
        { name:'WebKit built-in PDF', filename:'internal-pdf-viewer', description:'', length:2 },
      ];
      p.length = 5;
      return p;
    }
  });
})();
```

### 最优方案: puppeteer-core + 有头 Chrome

```javascript
const browser = await puppeteer.launch({
  executablePath: CHROME_PATH,
  headless: false,
  args: [
    '--disable-blink-features=AutomationControlled',
    '--window-position=-2000,-2000',
    '--no-first-run', '--disable-sync',
    '--lang=zh-CN', '--window-size=1920,1080',
  ],
  ignoreDefaultArgs: ['--enable-automation'],
});
```

## 通用策略: 逆向前的侦察清单

在投入大量时间逆向签名算法之前，按此顺序检查:

1. **拦截所有 API 请求**: 用 puppeteer setRequestInterception 或 jshook network_enable 抓取页面加载时的全部请求
2. **区分必需签名和非必需签名的端点**: 很多站点有不需要签名的备用/主力端点
3. **确认目标端点在真浏览器中是否正常**: 如果真浏览器也失败，说明端点本身有问题
4. **检查请求头差异**: 注意 `tmapi-client`、`appfrom`、`cluster` 等容易被忽略的业务头
5. **确认 cookie 链**: 会话初始化 -> set-cookie -> 后续请求带回

## 用户强烈偏好

- **纯协议实现优先** -- "不能用 Playwright 自动化"、"我要协议不要模拟"
- 必须是算法逆向，不是浏览器自动化
- 可以扣 JS 代码补环境用 py 运行
- 如果纯协议不可行，用 headed Chrome + puppeteer-core (窗口隐藏) 作为次优方案
- 如果有图形验证码，协议获取图片让用户手动填入
- sunny 抓包数据作为分析输入
- **逆向前先确认目标接口可行性，避免无效劳动**

## 常见加密函数 Hook 目标

```javascript
// 用 jshook 的 AI Hook 生成器监控
// 目标：window.btoa, window.atob
// 目标：crypto.subtle.encrypt/decrypt
// 目标：CryptoJS.AES/MD5/SHA
// 目标：XMLHttpRequest.send (拦截请求参数)
// 目标：fetch (拦截 API 调用)
```

## Token 链分析

很多站点的认证流程是多步 token 链：
1. 首次请求获取初始 token (validate/csrf)
2. 对 token 进行变换（编码/加密/拼接）
3. 带变换后的 token 请求下一步
4. 最终获得认证 cookie/session

分析时需要完整跟踪每一步的输入输出。

## 浏览器指纹监控模式 (JSHook MCP)

- 当需要系统分析 Canvas/Network/Cookie/Navigator/Screen/WebGL/Audio 等指纹向量时，可在 jshook MCP 中集成专用指纹监控模块，通过单个注入脚本统一 Hook 相关 API。
- 注入脚本通过 `Function` 包装、`Object.defineProperty` 等方式拦截常见调用，并在页面中挂载全局 `window.__FP_MONITOR__`，提供 `getLogs(filter)` / `getReport()` / `export()` / `clear()` / `getStats()` 等方法。
- MCP 侧暴露 5 个配套工具：`fp_inject`（在 `page_navigate` 前注入脚本）、`fp_get_report`（获取统计报告）、`fp_get_logs`（按接口/关键字过滤日志）、`fp_export`（导出全部原始日志）、`fp_clear`（清除监控数据），实现与桌面版指纹分析器等价的能力。
- 推荐调用顺序：`browser_launch` → 可选 `stealth_inject` → `fp_inject` → `page_navigate` 加载目标站点 → 根据需要调用 `fp_get_report`/`fp_get_logs`/`fp_export`，始终确保注入发生在页面执行前。

## Web 动态列表一次性采集 + 本地可搜索下拉 (Tinder 国家列表)

- 对于页面内可搜索的长列表（如 Tinder 手机号弹窗中的国家选择器），优先用 Playwright/浏览器自动化一次性采集 DOM 中的全部项，并落盘为 JSON（如 `tinder_countries.json`，字段包含 `name` / `dial_code` / `display`）。
- 业务层新建 `countries.py` 等模块负责加载 JSON，提供「页面国家名/区号 ↔ 第三方服务代码（如 5sim 国家代码）」的映射，并给 GUI 暴露完整的 display 列表。
- GUI 端用 `QComboBox + QCompleter` 或输入框 + 下拉列表构建可搜索下拉，支持按国家名、区号搜索；选中的 Tinder 国家用于自动化填表，对应的 5sim 代码通过映射表传给接码接口。
- 若采集文件缺失或为空，则回退到原有的静态国家列表配置，保证老配置与脚本仍可正常运行。

## SunnyNet 代理在逆向中的应用

| 功能 | 用途 |
|------|------|
| 响应文件替换 | 用本地修改过的 JS 替换 CDN 原文件 |
| String(UTF8) 替换 | 全局修改 HTTP 请求/响应中的字符串 (如 UA) |
| 断点拦截 | URL 正则匹配自动拦截请求，可修改请求体/响应体 |
| request_search | 按 URL/方法/状态码搜索已抓取的请求 |
| request_get | 查看完整请求头、请求体、响应头、响应体 |

### SunnyNet 操作注意

- 替换规则对本机所有流量生效，PowerShell 下载文件也会被拦截
- 添加响应文件规则前确保本地文件是最新修改版
- 分析完成后及时清除不需要的规则，避免干扰正常上网

Last updated: 2026-03-07
