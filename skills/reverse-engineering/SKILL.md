---
name: reverse-engineering
description: JavaScript 和 Web 应用逆向工程指南。用于分析加密算法、绕过反调试、提取 Webpack 模块、Hook 浏览器 API、分析混淆代码。适用于安全研究和爬虫开发。包含 AntiDebug Breaker 插件的全部 Hook 技术。
---

# JavaScript 逆向工程指南

## 浏览器调试技巧

### 常用断点类型

1. **XHR/Fetch 断点**: 监控网络请求 → DevTools → Sources → XHR/fetch Breakpoints
2. **DOM 断点**: 右键元素 → Break on → subtree modifications
3. **事件监听断点**: DevTools → Sources → Event Listener Breakpoints
4. **条件断点**: 右键行号 → Add conditional breakpoint

### 控制台技巧

```javascript
getEventListeners($0)          // 查找元素绑定的事件
monitor(functionName)          // 监控函数调用
copy(someObject)               // 复制到剪贴板
$_                             // 获取最后一个表达式的值
$('selector')                  // document.querySelector
$$('selector')                 // document.querySelectorAll
```

### 推荐浏览器扩展

- **AntiDebug Breaker** (https://github.com/0xsdeo/AntiDebug_Breaker): 一键过反调试 + Hook 加密库 + Vue DevTools 激活
- **Tampermonkey**: 用户脚本管理
- **Resource Override**: 替换网页资源
- **EditThisCookie**: Cookie 管理

---

## 反调试绕过 (来自 AntiDebug Breaker)

### Bypass Debugger - 过无限 debugger

原理：Hook `eval`、`Function` 构造器和 `Function.prototype.constructor`，将所有 `debugger` 字符串替换为空。

```javascript
(function() {
    'use strict';
    let temp_eval = eval;
    window.eval = function () {
        if (typeof arguments[0] == "string") {
            arguments[0] = arguments[0].replaceAll(/debugger/g, '');
        }
        return temp_eval(...arguments);
    }

    let Bypass_debugger = Function;
    Function = function () {
        for (let i = 0; i < arguments.length; i++) {
            if (typeof arguments[i] == "string") {
                arguments[i] = arguments[i].replaceAll(/debugger/g, '');
            }
        }
        return Bypass_debugger(...arguments);
    }
    Function.prototype = Bypass_debugger.prototype;

    Function.prototype.constructor = function () {
        for (let i = 0; i < arguments.length; i++) {
            if (typeof arguments[i] == "string") {
                arguments[i] = arguments[i].replaceAll(/debugger/g, '');
            }
        }
        return Bypass_debugger(...arguments);
    }
    Function.prototype.constructor.prototype = Function.prototype;
})();
```

### Hook Log - 防止 console 被重写

原理：用 Proxy 代理 console 对象，拦截 set 操作阻止方法被覆盖。同时防止 `window.console` 被整体替换。

```javascript
(function () {
    'use strict';
    const readonlyProps = ['log', 'trace', 'groupCollapsed', 'groupEnd'];
    const readonlyConsole = new Proxy(console, {
        set(t, k, v, r) {
            if (readonlyProps.includes(k)) {
                console.groupCollapsed(`%c有代码试图重写console.${k}方法，已阻止`,"color: #ff6348;", v);
                console.trace();
                console.groupEnd();
                return true;
            }
            return Reflect.set(t, k, v, r);
        }
    });
    Object.defineProperty(window, 'console', {
        configurable: true,
        enumerable: false,
        get: function () { return readonlyConsole; },
        set: function (v) {
            console.groupCollapsed("%c有代码试图重写console，已阻止", "color: #ff6348;", v);
            console.trace();
            console.groupEnd();
        }
    });
})();
```

### Hook Table - 过 console.table 时间差检测

原理：某些网站用 `console.table()` 的执行时间差来检测 DevTools 是否打开（DevTools 打开时 console.table 渲染慢）。

```javascript
console.table = function () {};
```

### Hook Clear - 禁止清除控制台

```javascript
console.clear = function() {};
```

### Hook Close - 防止关闭页面

```javascript
window.close = function() {};
```

### Hook History - 防止返回/跳转

```javascript
window.history.go = function() {};
window.history.back = function () {};
```

### Fixed Window Size - 窗口尺寸欺骗

原理：反调试通过 `outerWidth - innerWidth > threshold` 检测 DevTools 面板。固定返回值绕过。

```javascript
(function () {
    'use strict';
    let inner_height = 660, inner_width = 1366;
    let outer_height = 760, outer_width = 1400;

    let ihDesc = Object.getOwnPropertyDescriptor(window, "innerHeight");
    Object.defineProperty(window, "innerHeight", {
        get: () => inner_height,
        set: () => ihDesc.set.call(window, inner_height)
    });
    let iwDesc = Object.getOwnPropertyDescriptor(window, "innerWidth");
    Object.defineProperty(window, "innerWidth", {
        get: () => inner_width,
        set: () => iwDesc.set.call(window, inner_width)
    });
    let owDesc = Object.getOwnPropertyDescriptor(window, "outerWidth");
    Object.defineProperty(window, "outerWidth", {
        get: () => outer_width,
        set: () => owDesc.set.call(window, outer_width)
    });
    let ohDesc = Object.getOwnPropertyDescriptor(window, "outerHeight");
    Object.defineProperty(window, "outerHeight", {
        get: () => outer_height,
        set: () => ohDesc.set.call(window, outer_height)
    });
})();
```

### 反 Hook 检测 (AntiAnti_Hook)

原理：网站可能通过 `Function.prototype.toString` 检测函数是否被 Hook（被 Hook 的函数 toString 输出不再是 `[native code]`）。还可能通过创建 iframe 获取原始函数来绕过 Hook。

```javascript
// 伪装被 Hook 的函数的 toString 输出
let temp_toString = Function.prototype.toString;
Function.prototype.toString = function () {
    if (this === Function.prototype.toString) {
        return 'function toString() { [native code] }';
    }
    if (hookedFunctions.has(this)) {
        const funcName = hookedFunctions.get(this);
        return `function ${funcName}() { [native code] }`;
    }
    return temp_toString.apply(this, arguments);
};

// 防止 iframe 反 Hook：代理 iframe.contentWindow
Object.defineProperty(HTMLIFrameElement.prototype, "contentWindow", {
    get: function () {
        let iframe_window = originalGet.apply(this);
        return new Proxy(iframe_window, {
            get: function (target, property, receiver) {
                // 如果属性是被 Hook 的方法，返回主 window 的 Hook 版本
                if (hookedMethodNames.has(property)) return hookedFn;
                return Reflect.get(target, property, target);
            }
        });
    }
});
```

### 动态脚本拦截（反第三方反调试SDK）

原理：部分网站使用 Botion/GeeTest 等第三方 SDK 动态加载反调试脚本。通过拦截 `script.src` 赋值和 `appendChild` 阻止加载。

```javascript
(function(){
    var blockedDomains = ['botion.com','netshunt.com','botlabpro.com'];
    function isBlocked(url) {
        if (!url) return false;
        var s = String(url).toLowerCase();
        return blockedDomains.some(d => s.indexOf(d) !== -1);
    }
    var origSrcDesc = Object.getOwnPropertyDescriptor(HTMLScriptElement.prototype, 'src');
    Object.defineProperty(HTMLScriptElement.prototype, 'src', {
        set: function(val) {
            if (isBlocked(val)) { this.__blocked = true; return; }
            return origSrcDesc.set.call(this, val);
        },
        get: origSrcDesc.get, configurable: true
    });
    var origAC = Node.prototype.appendChild;
    Node.prototype.appendChild = function(child) {
        if (child && child.tagName === 'SCRIPT' && (child.__blocked || isBlocked(child.src)))
            return child;
        return origAC.call(this, child);
    };
})();
```

---

## 加密库 Hook (来自 AntiDebug Breaker)

### Hook CryptoJS - 对称加密/哈希/HMAC

原理：Hook `Function.prototype.apply`，通过检测参数特征（`ciphertext`、`key`、`iv`、`algorithm`、`mode`、`padding`、`blockSize`、`formatter`）识别 CryptoJS 加密调用，自动输出 key、iv、mode、padding、密文。

关键检测逻辑：

```javascript
// 对称加密特征检测
function hasEncryptProp(obj) {
    const requiredProps = ['ciphertext','key','iv','algorithm','mode','padding','blockSize','formatter'];
    return requiredProps.every(prop => prop in obj);
}

// 对称解密特征检测
function hasDecryptProp(obj) {
    return ['sigBytes','words'].every(prop => prop in obj);
}

// 哈希/HMAC 特征检测：原型链上有 $super + _doFinalize + finalize
// Hook finalize 方法输出明文和密文

let temp_apply = Function.prototype.apply;
Function.prototype.apply = function () {
    // 对称加密：arguments[1].length === 1 && hasEncryptProp(arguments[1][0])
    if (hasEncryptProp(arguments[1]?.[0])) {
        console.log("对称加密Hex key：", arguments[1][0]["key"].toString());
        console.log("对称加密Hex iv：", arguments[1][0]["iv"]?.toString());
        console.log("填充模式：", arguments[1][0]["padding"]);
        console.log("运算模式：", arguments[1][0]["mode"]?.["Encryptor"]?.["processBlock"]);
    }
    // 对称解密：arguments[1].length === 3 && hasDecryptProp(arguments[1][1])
    // 哈希/HMAC：原型链 _doFinalize + finalize -> Hook finalize
    return temp_apply.call(this, ...arguments);
}
```

### Hook JSEncrypt RSA

原理：Hook `Function.prototype.call`，通过检测参数原型链上是否有 RSA 特征属性（`getPublicKey`、`getPrivateKey`、`encrypt`、`decrypt`）来识别 JSEncrypt 调用。

```javascript
// RSA 特征检测
function hasRSAProp(obj) {
    const requiredProps = ['constructor','getPrivateBaseKey','getPrivateBaseKeyB64',
        'getPrivateKey','getPublicBaseKey','getPublicBaseKeyB64','getPublicKey',
        'parseKey','parsePropertiesFrom'];
    return requiredProps.every(prop => prop in obj);
}

let temp_call = Function.prototype.call;
Function.prototype.call = function () {
    if (arguments[0]?.__proto__ && hasRSAProp(arguments[0].__proto__)) {
        // Hook encrypt 方法
        arguments[0].__proto__.__proto__.encrypt = function () {
            let encrypt_text = originalEncrypt.bind(this, ...arguments)();
            console.log("RSA 公钥：", this.getPublicKey());
            console.log("RSA加密 原始数据：", ...arguments);
            console.log("RSA加密 Base64 密文：", hexToBase64(encrypt_text));
            return encrypt_text;
        }
    }
    return temp_call.bind(this, ...arguments)();
}
```

### Hook SM-crypto (国密算法 SM2/SM3/SM4)

原理：Hook `Function.prototype.call`，拦截 webpack 模块加载（`arguments.length === 4 && arguments[1]?.exports`），检测导出对象特征识别 SM2/SM3/SM4 函数。

```javascript
// SM2 特征：exports 有 doEncrypt/doDecrypt/doSignature/doVerifySignature/generateKeyPairHex
// SM4 特征：用已知测试向量验证 encrypt/decrypt 函数
// SM3 特征：函数 toString 包含 'invalid mode' 且测试输出匹配已知哈希值

const raw_call = Function.prototype.call;
Function.prototype.call = function() {
    const result = Reflect.apply(raw_call, this, arguments);
    if (arguments.length === 4 && arguments[1]?.exports) {
        const exports = arguments[1].exports;
        if (exports.doEncrypt && has_SM2_Prop(exports)) {
            // Hook SM2 加密，输出明文、公钥、密文
        }
        if (exports.encrypt && sm4_encrypt_test(exports.encrypt) === "known_ciphertext") {
            // Hook SM4 加密，输出明文、key、iv、模式、密文
        }
        if (typeof exports === "function" && sm3_encrypt_test(exports) === "known_hash") {
            // Hook SM3 哈希，输出明文和密文
        }
    }
    return result;
}
```

---

## Hook 技术

### 基础 Hook 模式

```javascript
(function() {
    var original = window.targetFunction;
    window.targetFunction = function() {
        console.log('[Hook] 参数:', arguments);
        var result = original.apply(this, arguments);
        console.log('[Hook] 返回值:', result);
        return result;
    };
})();
```

### 常用 Hook 目标

```javascript
// XHR Hook
var _open = XMLHttpRequest.prototype.open;
XMLHttpRequest.prototype.open = function(method, url) {
    console.log('[XHR]', method, url);
    return _open.apply(this, arguments);
};

// Fetch Hook
var _fetch = window.fetch;
window.fetch = function(url, options) {
    console.log('[Fetch]', url, options);
    return _fetch.apply(this, arguments);
};

// JSON Hook
var _stringify = JSON.stringify;
JSON.stringify = function(obj) {
    console.log('[JSON.stringify]', obj);
    return _stringify.apply(this, arguments);
};

// Cookie Hook
Object.defineProperty(document, 'cookie', {
    get: function() { console.log('[Cookie] 读取'); return ''; },
    set: function(val) { console.log('[Cookie] 设置:', val); }
});
```

---

## Vue DevTools 强制激活

原理：监听 Vue DevTools 消息事件，找到 Vue 根实例后强制启用 devtools，支持 Vue 2 和 Vue 3 + Pinia。

```javascript
// Vue 2 激活
function crackVue2() {
    const app = document.querySelector('*').__vue__;
    let Vue = Object.getPrototypeOf(app).constructor;
    while (Vue.super) Vue = Vue.super;
    Vue.config.devtools = true;
    window.__VUE_DEVTOOLS_GLOBAL_HOOK__.emit('init', Vue);
}

// Vue 3 激活
function crackVue3() {
    const app = document.querySelector('*').__vue_app__;
    const devtools = window.__VUE_DEVTOOLS_GLOBAL_HOOK__;
    devtools.enabled = true;
    devtools.emit('app:init', app, app.version, {
        Fragment: Symbol.for('v-fgt'),
        Text: Symbol.for('v-txt'),
        Comment: Symbol.for('v-cmt'),
        Static: Symbol.for('v-stc'),
    });
    // 同时激活 Pinia DevTools
    const pinia = app.config.globalProperties.$pinia;
    if (pinia) { pinia.use(devtoolsPlugin); registerPiniaDevtools(app, pinia); }
}
```

---

## 加密算法识别

| 算法 | 特征 |
|------|------|
| MD5 | 32位十六进制，包含 `0123456789abcdef` |
| SHA1 | 40位十六进制 |
| SHA256 | 64位十六进制 |
| Base64 | 包含 `=` 填充，字符集 `A-Za-z0-9+/` |
| AES | 通常配合 IV 使用，输出为 Base64 或 Hex |
| RSA | 公钥加密，输出较长 |
| SM2 | 国密非对称，特征函数 `doEncrypt`/`doDecrypt` |
| SM3 | 国密哈希，64位十六进制 |
| SM4 | 国密对称，特征函数名含 `sm4`，支持 CBC/ECB |

---

## Webpack 模块提取

### 定位 Webpack 入口

1. 搜索 `webpackJsonp` 或 `webpackChunk`
2. 搜索 `__webpack_require__`
3. 查找 IIFE 包装的大型对象

### 提取模块

```javascript
// Webpack 4
var modules = {};
webpackJsonp.push([['_'], {'_': function(m, e, r) { modules = r.c; }}, [['_']]]);

// Webpack 5
var modules = {};
webpackChunk.push([['_'], {'_': function(m, e, r) { modules = r.c; }}, [['_']]]);
```

---

## 混淆代码分析

### 常见混淆类型

1. **变量名混淆**: `_0x1234`, `$_IJDD`
2. **字符串加密**: XOR + decodeURI（Botion/GeeTest 风格）
3. **控制流平坦化**: switch-case 状态机
4. **死代码注入**: 无用分支干扰
5. **iframe 反 Hook**: 通过 iframe 获取原始函数

### GeeTest/Botion SDK 混淆特征

- 命名空间如 `FqAkO`，变量名 `$_XXXX`
- 字符串通过 `decodeURI` + XOR 解密
- switch-case 状态机控制流
- 动态加载脚本从 `static.botion.com` / `bcaptcha.botion.com`

---

## 调试流程

1. **确定目标**: 找到需要分析的接口或功能
2. **开启 AntiDebug Breaker**: Bypass Debugger + Hook Log + Hook Table
3. **设置断点**: XHR 断点捕获请求
4. **开启加密 Hook**: Hook CryptoJS / JSEncrypt / SM-crypto
5. **回溯调用栈**: 找到加密入口
6. **分析算法**: 单步调试理解逻辑
7. **提取代码**: 复制关键函数
8. **本地测试**: 验证提取的代码
9. **封装接口**: 打包成可复用模块

---

## 实用工具

### 在线工具

- **AST Explorer**: 分析 JavaScript AST
- **beautifier.io**: 代码格式化
- **deobfuscate.io**: 自动反混淆
- **CyberChef**: 多功能编解码
- **Fuzz Crypto Algorithms**: https://github.com/0xsdeo/Fuzz_Crypto_Algorithms

### 抓包工具

- **Fiddler**: Windows HTTP 代理
- **Charles**: 跨平台代理
- **mitmproxy**: 可编程代理
- **Wireshark**: 网络包分析
