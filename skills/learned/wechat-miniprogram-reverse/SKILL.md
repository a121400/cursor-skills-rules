---
name: wechat-miniprogram-reverse
description: 微信小程序逆向与环境绕过。PC 端小程序 H5 页面环境检测绕过、WMPFDebugger 开 CDP 端口 + jshook 连接逆向、SunnyNet JS 替换、User-Agent 伪装、Vue 路由分析、ucToken 用户链路调试。适用于小程序签到/抢购/自动化场景。
---

# 微信小程序 H5 逆向 (实战总结)

## 适用场景

微信小程序内嵌 H5 页面的逆向分析，在 PC 微信中运行并绕过移动端环境检测，实现自动签到/抢购等。

## 核心工具链

| 工具 | 用途 |
|------|------|
| **WMPFDebugger** | 在 PC 微信内打开小程序并暴露 CDP 端口，供 jshook 连接 |
| **jshook MCP** | 连接 WMPFDebugger 暴露的 CDP 端口，做脚本注入/断点/抓包等逆向 |
| SunnyNet 代理 | 抓包 + JS 响应文件替换 + 协议头改写 |
| WeChatAppEx (PC 微信) | 小程序运行环境 |
| Node.js | 扣代码执行加密算法 |

## 整体流程

```
1. SunnyNet 抓包分析小程序 H5 页面的 JS/API 请求
2. 下载目标 JS 文件到本地
3. 分析 JS 中的环境检测逻辑 (created/onShow 生命周期)
4. 最小化修改: 绕过 PC 检测 + UA 伪装
5. SunnyNet 添加 "响应文件" 规则替换 JS
6. SunnyNet 添加 "String(UTF8)" 规则改写 HTTP 协议头
7. PC 微信刷新页面验证
```

## 环境检测绕过 (关键)

### 典型检测模式

小程序 H5 页面常见的 Vue 组件检测:

```javascript
// created() 中判断是否移动端
created(){
  if(Object(r["l"])()){   // isMobile() 检测
    this.handleSession();
    this.onShow();
  } else {
    this.tipWords = "请用手机打开参与活动";
  }
}
```

**绕过方法**: 将 `if(Object(r["l"])())` 改为 `if(!0)`

### UA 伪装 (JS 层面)

在 `created()` 最前面注入:

```javascript
created(){
  try{
    Object.defineProperty(navigator,"userAgent",{get:function(){
      return"Mozilla/5.0 (Linux; Android 13; SM-G9980) AppleWebKit/537.36 Chrome/132.0.0.0 Mobile Safari/537.36 MicroMessenger/8.0.44.2502(0x28002C37) NetType/WIFI Language/zh_CN miniProgram/wx目标APPID"
    }});
    Object.defineProperty(navigator,"platform",{get:function(){return"Linux armv8l"}});
  }catch(e){}
  if(!0){this.handleSession();this.onShow();/* 原始逻辑 */}
}
```

### SunnyNet 协议头替换

JS 层面的 UA 修改不影响 HTTP 请求头，必须配合 SunnyNet 的 `String(UTF8)` 替换:

```
规则1: "Windows NT 10.0; Win64; x64"  ->  "Linux; Android 13; SM-G9980"
规则2: "MiniProgramEnv/Windows WindowsWechat/WMPF WindowsWechat(0x63090a13) UnifiedPCWindowsWechat(0xf2541211)"  ->  "MiniProgramEnv/android"
```

## SunnyNet 替换规则操作

### JS 响应文件替换

```
类型: 响应文件
源: login.542bceb1.js         (文件名匹配，不需要完整路径)
目标: C:\path\to\hook\login.542bceb1.js  (本地修改后的文件)
```

**注意**: SunnyNet 启动后，本机所有 HTTP 请求都经过代理。用 PowerShell 下载文件时也会被拦截，需要先清除替换规则再下载原始文件。

### 常见陷阱

1. **缓存问题**: SunnyNet 可能缓存响应，修改本地文件后需等 SunnyNet 刷新
2. **下载被拦截**: 用 `Invoke-WebRequest` 下载原始 JS 前必须先清除响应文件规则
3. **改动要最小**: 只改检测位置和 UA，保留原始业务逻辑。过度修改会导致路由/数据异常

## Vue 路由分析

### 路由映射

小程序 H5 常用 Vue Router hash 模式 (`#/login`, `#/homePage`)。路径映射在常量中:

```javascript
// c["e"][module] 映射 module 编号到路由路径
// module "4" -> "homePage"
// module "7" -> "machine"
```

### 关键生命周期

```
created()  -> handleSession() 初始化 sessionStorage
           -> onShow() 主逻辑入口
onShow()   -> getCompleteActivityInfo() 获取活动信息
           -> validateEnvByRootCode() 环境/根码验证
           -> validateActivityStatus() 活动状态验证
           -> _getPageData() 页面数据
           -> env 检测 (Object(r["h"])())
           -> getUserInfoByModule() 根据 module 获取用户信息
           -> goHomePage() 导航到主页
```

### 用户信息链路

```
ucToken (URL 参数) -> getUserInfoByUcToken (API) -> sessionStorage["userInfo"]
                   -> updateAndRedirect -> goHomePage
```

签到/操作失败时检查: sessionStorage 中的 `userInfo` 是否有 `uid/openId/unionId`。如果 `userKey` 仍为 `"TEST_"` 说明用户信息未解析。

## Hook 流程 (WMPFDebugger + jshook)

1. **WMPFDebugger**: 在 PC 微信里打开目标小程序，在工具中开启/暴露 CDP (Chrome DevTools Protocol) 调试端口。
2. **jshook MCP**: 配置连接上述 CDP 端口（如 `ws://127.0.0.1:xxxx`），连接成功后即可用 jshook 的 collect_code、断点、Hook、network 等能力对小程序页面做逆向。
3. 小结：先用 WMPFDebugger 打开 CDP 端口，再用 jshook 连上该端口进行逆向。

## PC 端「请使用手机微信打开」绕过（WMPF 平台检测）

### 分析阶段

- **jshook**：browser_launch 后 ai_hook_generate，target 为 wx.getSystemInfoSync，captureReturn/captureStack，确认是否依赖 platform 字段。
- **Sunny**：若接口在 UA 含 WindowsWechat/WMPF 时返回错误码，则为服务端检测，用替换规则改 UA；若接口返回成功但页面仍提示「请使用手机微信打开」，则为**客户端检测**，需 CDP 注入覆盖 wx。

### 客户端绕过（CDP 注入）

- **必须用 Object.defineProperty**：wx 的 getSystemInfoSync 等是 getter/setter，直接赋值无效。对 wx.getSystemInfoSync、getSystemInfo、getDeviceInfo 用 defineProperty 包一层，在返回值里把 platform 改为 'android'，system/brand/model 一并改为真机值。
- **持久化**：新页面会重新加载 JS，覆盖会丢。用 **Page.addScriptToEvaluateOnNewDocument** 注入一段脚本，脚本内用 **setInterval** 轮询检测 `typeof wx !== 'undefined'` 后执行 patchWx(wx)。**不要**用 globalThis 的 setter trap 拦截 wx，会丢原始函数引用导致 apply 报错。
- **注入后**：调用 wx.reLaunch({ url: '/入口页' })（入口从 __wxConfig.entryPagePath 或 pages[0] 取，去掉 .html）。多个 context（appContext、page-frame）都需 patch，先找 typeof getCurrentPages === 'function' 的 appContext。
- **若之前加过错误持久化脚本**：先 Page.removeScriptToEvaluateOnNewDocument(1..3) 再加新的，避免旧 setter trap 残留。

### 服务端 UA 替换（Sunny）

- 若后端根据 User-Agent 拒绝：String(UTF8) 规则把 "MiniProgramEnv/Windows WindowsWechat/WMPF" 替换为 "MiniProgramEnv/Android ..."。

## 调试技巧

### SunnyNet 请求分析

```
1. request_search(url="关键词") 搜索目标 API
2. request_get(theology=ID) 查看请求/响应详情
3. 关注 headParams.userKey 是否为空 (用户身份未建立)
4. 关注 response body 的 code 字段 (非 "000000" 表示异常)
```

### 常见错误码

| 错误 | 含义 | 排查方向 |
|------|------|----------|
| 签到异常 (A80065) | 用户身份未建立 | 检查 userInfo 链路 |
| 请用手机打开 | PC 环境被检测 | 检查 isMobile() 绕过 |
| 活动已结束 | actvStatus 异常 | 检查 validateActivityStatus |
| 跳转 /transfer | API 返回非 000000 | 检查 getCompleteActivityInfo |

## 修改原则

1. **先下载原始 JS**: 清除 SunnyNet 规则 -> 下载 -> 再加规则
2. **只改两处**: PC 检测绕过 (`if(!0)`) + UA 伪装 (`Object.defineProperty`)
3. **保留原始流程**: 不要跳过 env 检测、不要跳过 getUserInfoByModule
4. **配合协议头**: JS 层改 navigator + SunnyNet 改 HTTP 头，双管齐下

Last updated: 2026-03-07
