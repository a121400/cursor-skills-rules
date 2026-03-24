---
name: cursor-plans-transcripts
description: Cursor plans 与对话数据总结工作流。遍历 plans/agent-transcripts 抽取可复用模式，决策更新或新建 skill/rules。当用户要求「总结 plans」「更新 skill」「整理对话经验」时使用。
---

# Cursor Plans 与对话数据 -> Skills/Rules 更新

## Plans 主题聚类速查（按文件名/内容）

- **发单/登录/TCP**：超时、并发、重试、线程池、重连卡死、ETIMEDOUT、网关黑名单
- **签名服务**：Rust/Python 接口、unidbg、headless emulator、局域网 APK、Frida 桥接 MCP
- **代理**：proxy_pool、SOCKS5、过期 TTL、限频 refresh、定时前预取
- **游戏/协议**：TCP 抓包、多游戏配置 game_profiles、迷宫/三国杀/天龙八部、协议解包组包
- **MCP**：看雪搜索、Frida 桥接、Sunny/tlbb 抓包、小程序调试、jshook、CDP
- **抢购/桌面**：嘉立创、天安门抢票、小乔有礼、币安、Tinder、Qt/Rust/C# GUI
- **接码/打码**：豪猪码、5sim、图鉴账号
- **文档/业务**：档案管理、按项目拆分账单 PDF、Excel 台账
- **Skills/Rules 自身**：cursor_skills 优化、batch-process-plans、fix_hooks 自动更新、plans-skills-sync

## 已有 skill 覆盖（总结时优先更新，不重复新建）

| 主题 | 对应 skill |
|------|------------|
| 看雪检索、逆向案例搜索 | kanxue-reverse-search |
| 抢购、代理池、币安、TLS、登录超时重连 | rush-buy-automation |
| MCP、抓包、协议、多游戏配置、Frida 桥接 | mcp-analysis-workflow |
| 小程序、F12、CDP、WMPF | wechat-miniprogram-reverse |
| 鲸探签名、Frida/unidbg | antfans-sms-sign |
| 顶象滑块 | dingxiang-slider-captcha |
| 豪猪码接码 | haozhuma-sms-api |
| 文档 Excel/PDF/Word、台账 | doc-processing |
| 逆向文档、工具链 | re-docs-index |
| JS 逆向、扣代码、RPC | js-reverse-practical |
| Web 反检测、指纹 | web-re-antidetect |
| Python 环境、依赖 | python-env-compat |
| Win GUI、pywin32、OCR | win-gui-automation |
| ADB、MuMu、Frida 移动端 | adb-emulator |
| EPL 语言 | epl-language |
| 计划与对话总结流程 | cursor-plans-transcripts |
| 安卓加固/模拟器/Frida 检测绕过 | android-anti-detect-bypass |
| JS AST 反混淆(Babel) | ast-deobfuscate |
| 腾讯防水墙滑块 | tcaptcha-solver |
| 数美(Shumei)点选验证码、fverify DES多Key | shumei-captcha-reverse |
| WebSocket 二进制协议逆向(Protobuf+AES) | ws-binary-protocol-reverse |
| 连信APP逆向、libzhangxin unidbg签名解密、EncryptedJsonRequest | lianxin-app-reverse |

## 数据位置

| 类型 | 路径 | 说明 |
|------|------|------|
| Plans | `C:\Users\Administrator\.cursor\plans\*.plan.md` | 大量 .plan.md，文件名多为「中文描述_短hash.plan.md」 |
| 对话 | `C:\Users\Administrator\.cursor\projects\<项目名>\agent-transcripts\` | 子目录为会话 UUID，内容多为 `.txt` 纯文本（Cursor 当前格式） |
| 已处理记录 | `~/.cursor/skills/learned/_processed.txt` | 每行一个已处理的 transcript 路径，用于去重 |
| Skills | `~/.cursor/skills/learned/<name>/SKILL.md` | 个人 learned 技能 |
| Rules | `~/.cursor/rules/*.mdc` | 按需引入的规则（如 reverse-engineering.mdc、git-workflow.mdc） |

## 执行流程（总结并更新时）

1. **收集**
   - 遍历 `plans\*.plan.md`，按文件名/内容做主题聚类（逆向、抢购、MCP、小程序、签名服务、抓包、GUI 等）。
   - 若需用到对话：在 `projects/*/agent-transcripts` 下找 `.txt`，排除 `_processed.txt` 中已列路径；按需读最近若干条。

2. **抽取**
   - 从 plan/对话中抽取：可复用工作流、工具组合、目录/命名约定、常见坑与解决方式。
   - 仅保留在 2+ 处出现或明显可复用的模式，避免为一次性方案新建 skill。

3. **决策**
   - 已有同类 skill（如 kanxue-reverse-search、rush-buy-automation、mcp-analysis-workflow）-> 在其 SKILL.md 中追加小节或更新。
   - 新主题且与现有差异大 -> 新建 `learned/<name>/SKILL.md`，遵循 create-skill 的 frontmatter 与篇幅要求。

4. **落笔**
   - 写 SKILL.md：简洁、步骤导向，写清触发场景与关键步骤。
   - 若需新规则：在 `rules/` 下新增 `.mdc`，frontmatter 中 `alwaysApply: false`，description 写明何时由 AI 引入。

5. **同步（可选）**
   - 若有 ~/.codex/skills/learned，用 PowerShell `Copy-Item -Force` 将对应 SKILL.md 同步过去。

## 与现有流程的关系

- **batch-process-plans-into-skills**：plans 批量分析、聚类、候选模式、再写/更新 skill 的完整规划已在该 plan 中；本 skill 是执行时的路径与步骤速查。
- **fix_hooks_auto-update_skill**：continuous-learning 用 `evaluate-session.ps1` 解析 `.txt` transcript，结果写 `_pending/`、更新 workflow-patterns；transcript 格式为纯文本（user:/assistant:、[Thinking]、[Tool call]），非 JSON。

## 注意

- 不主动生成 Markdown 总结文档，除非用户明确要求。
- 技能描述用第三人称，包含「做什么 + 何时用」；单 SKILL.md 建议控制在 500 行内。
