---
name: re-docs-index
description: 逆向工程文档与教程索引。进行安卓逆向、JS 逆向、小程序逆向或 PC 游戏逆向（Cheat Engine、x64dbg、IDA）时，从此技能中查找官方文档、入门教程与工具配合说明。
---

# 逆向工程文档索引

进行 Android RE、JS RE、小程序逆向、PC 游戏逆向时，优先从下表查找官方文档与教程链接；需要扩展时可结合项目内 `docs/逆向文档索引.md`。

## 一、安卓逆向 (Android RE)

**入门与教程**：Android App RE 101 (maddiestone.github.io/AndroidAppRE)、看雪学苑安卓逆向入门、crifan 电子书、百度/腾讯云入门流程。

**工具与仓库**：JADX (github.com/skylot/jadx)、apk.sh、AndroidAppRE 仓库。

**速查**：反编译/回编译 APKTool；查看源码 Jadx/JD-GUI；动态调试 IDA、JEB、Frida。

| 名称 | 链接 |
|-----|------|
| Android App RE 101 | https://maddiestone.github.io/AndroidAppRE/ |
| 看雪 - 安卓逆向入门 | https://www.kanxue.com/book-leaflet-96.htm |
| crifan 安卓安全与逆向 | https://book.crifan.org/books/android_app_security_crack/website/ |
| JADX | https://github.com/skylot/jadx |
| apk.sh | https://github.com/niclaslindqvist/apk.sh |

## 二、JS 逆向 (JavaScript RE)

**教程**：Restore-JS、腾讯云 JS 逆向爬虫、PingCode 流程、掘金混淆案例。

**仓库**：Crack-JS-Spider、js_reverse、decodeObfuscator、AST 转 JS 源码。

**技术点**：找入口（Network Initiator、XHR/fetch 断点）；Hook（Tampermonkey、btoa/encrypt）；还原（AST 反混淆、Wasm）。

| 名称 | 链接 |
|-----|------|
| Restore-JS | https://github.com/LoseNine/Restore-JS |
| 腾讯云 JS 逆向实战 | https://cloud.tencent.com/developer/article/2528493 |
| Crack-JS-Spider | https://github.com/LoseNine/Crack-JS-Spider |
| decodeObfuscator | https://github.com/Tsaiboss/decodeObfuscator |

## 三、小程序逆向 (微信等)

**教程**：腾讯云微信小程序反编译、反编译一篇够用、devtool 抓包+wxapkg、secevery 逆向步骤。

**路径**：安卓 `/data/data/com.tencent.mm/MicroMsg/[32位hex]/appbrand/pkg/`；Windows `WeChat Files\Applet\{AppID}\`。

**工具**：wxappUnpacker、wxapkg；依赖 esprima、css-tree、vm2、uglify-es、js-beautify 等。

| 名称 | 链接 |
|-----|------|
| 腾讯云 - 微信小程序反编译解包 | https://cloud.tencent.com/developer/article/2220042 |
| 腾讯云 - 反编译小程序一篇够用 | https://cloud.tencent.cn/developer/article/1582324 |
| wxapkg 反编译 (oacia) | https://oacia.dev/wechat-mini-program-with-devtool/ |

## 四、PC 游戏逆向 (CE / x64dbg / IDA)

### Cheat Engine

| 名称 | 链接 |
|-----|------|
| CE 官网 | https://www.cheatengine.org/ |
| CE 官方教程 | https://www.cheatengine.org/tutorials.php |
| CE Wiki | http://wiki.cheatengine.org/ |
| CE 源码 | https://github.com/cheat-engine/cheat-engine |
| 腾讯云 CE 教程汉化 | https://cloud.tencent.com/developer/article/2202022 |
| Ganlv CE 教程 | https://ganlvtech.github.io/2018/01/25/cheat-engine-tutorial/ |

### x64dbg

| 名称 | 链接 |
|-----|------|
| x64dbg 官网 | https://x64dbg.com/ |
| 官方文档 | https://help.x64dbg.com/en/latest/ |
| 中文站 | https://www.x64dbg.cn/ |
| 插件 Wiki | https://github.com/x64dbg/x64dbg/wiki/Plugins |

### IDA

| 名称 | 链接 |
|-----|------|
| Hex-Rays 文档 | https://docs.hex-rays.com/ |
| IDA Pro 中文网 - 教程 | https://www.idaprocn.cn/support.html |
| IDA Pro 中文网 - 技巧 | https://www.idapro.net.cn/jiqiao/ |
| 周哥教IT 逆向分析 | https://www.zhougejiaoit.com/lessons/ida/ida-0.html |

### 工具配合

- **CE + x64dbg**：CE 扫描/锁定地址 →「找出是什么改写了/访问了」得指令 → x64dbg 下断、改指令或补丁。
- **CE + IDA**：CE 定位数据与代码地址 → IDA 静态分析基址与偏移 → CE 注入或 Trainer。
- **x64dbg + IDA**：IDA 静态分析 → x64dbg 附加调试；Scylla 做导入表修复。
- **CE 常用步骤**：精确/未知值扫描 → 找出改写/访问 → 指针与多级指针 → 代码注入/自动汇编 → Trainer 或 Lua。

## 使用说明

- 链接以当前可访问为准，失效可搜索标题或作者名。
- 仅做授权内的安全研究，遵守法律法规与平台条款。
