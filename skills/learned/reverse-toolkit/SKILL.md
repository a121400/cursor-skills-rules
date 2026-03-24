---
name: reverse-toolkit
description: 逆向工程通用工具箱。执行原则(必须实际执行/测试/诊断/重试)、IDA/JADX/CE/x64dbg 协作流程、看雪/吾爱论坛搜索、逆向文档索引。所有逆向任务(JS/APP/验证码/协议/PC游戏)自动引入。
---

# 逆向工程通用工具箱

## 一、执行原则（所有逆向场景强制遵循）

### 适用范围

所有逆向相关场景：安卓/模拟器、JS 逆向、验证码、协议分析、PC 逆向（CE/x64dbg/IDA）、小程序等。

### 核心规则

1. **必须实际执行**：不满足于给出方案，直接执行命令/脚本/工具并观察输出。
2. **失败时诊断再决定**：根据报错跑 1~3 条诊断命令，根据结果重试或调整。
3. **与用户交互解决问题**：仅在确实需要用户本机操作时暂停，明确一条事项。
4. **没解决别停**：禁止以「已给建议」「可选方案」收尾；必须继续换思路、重试。

## 二、IDA/JADX/CE/x64dbg 协作流程

逆向需附加/加载目标文件时：

1. **要求目标文件进项目目录**：告诉用户把待分析文件（.exe/.dll/.apk/.so/.dex）放到当前项目根目录或 `target/` 子目录。
2. **请用户在本机附加**：IDA/JADX/x64dbg/CE 需在本机运行，请用户打开文件或附加进程。
3. **后续协作**：用户提供函数名、地址、截图、伪代码后，AI 继续给出思路或写脚本。

### 工具配合

- **CE + x64dbg**：CE 扫描/锁定地址 → 找出改写指令 → x64dbg 下断、改指令或补丁
- **CE + IDA**：CE 定位地址 → IDA 静态分析基址与偏移 → CE 注入或 Trainer
- **x64dbg + IDA**：IDA 静态分析 → x64dbg 附加调试；Scylla 做导入表修复

## 三、论坛搜索（看雪/吾爱破解）

依赖 `reverse-forum-search` MCP。逆向时自动检索类似案例。

### 触发方式

- **显式**：消息以 `【看雪搜索】`、`【吾爱搜索】` 或 `[逆向搜索]` 开头
- **隐式**：描述中同时出现 2+ 要素（平台词 + 关键点）时触发一次搜索

### 执行流程

1. 抽取检索意图，组合 1~3 条 query
2. 对每条 query 调用 `reverse_forum_search`（默认双论坛都搜）
3. 挑选代表性帖子 1~3 篇，`reverse_forum_thread_detail` 抓详情
4. 输出「逆向思路总结」：可参考帖子、可复用步骤、工具链、常见坑

### 预算

每轮最多：搜索 3 query × 2 论坛、详情 3 篇、看雪回复解锁 2 篇。

## 四、逆向文档索引

### 安卓逆向

| 名称 | 链接 |
|-----|------|
| Android App RE 101 | https://maddiestone.github.io/AndroidAppRE/ |
| 看雪安卓逆向入门 | https://www.kanxue.com/book-leaflet-96.htm |
| JADX | https://github.com/skylot/jadx |

### JS 逆向

| 名称 | 链接 |
|-----|------|
| Restore-JS | https://github.com/LoseNine/Restore-JS |
| 腾讯云 JS 逆向实战 | https://cloud.tencent.com/developer/article/2528493 |
| Crack-JS-Spider | https://github.com/LoseNine/Crack-JS-Spider |

### 小程序逆向

| 名称 | 链接 |
|-----|------|
| 微信小程序反编译 | https://cloud.tencent.com/developer/article/2220042 |
| wxapkg 反编译 | https://oacia.dev/wechat-mini-program-with-devtool/ |

### PC 逆向

| 工具 | 官网/文档 |
|------|----------|
| Cheat Engine | https://www.cheatengine.org/ |
| x64dbg | https://help.x64dbg.com/en/latest/ |
| IDA | https://docs.hex-rays.com/ |

### 工具配合速查

- CE 步骤：精确/未知值扫描 → 找出改写/访问 → 指针与多级指针 → 代码注入 → Trainer/Lua
- 安卓路径：`/data/data/com.tencent.mm/MicroMsg/[hex]/appbrand/pkg/`
- 小程序工具：wxappUnpacker、esprima、css-tree
