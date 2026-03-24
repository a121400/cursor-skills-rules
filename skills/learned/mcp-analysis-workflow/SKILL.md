---
name: mcp-analysis-workflow
description: MCP 工具分析工作流。游戏协议抓包解析、多游戏 profile 配置、Frida 桥接 MCP(无CDP时)、SunnyNet HTTP API 抓取、加密参数对照分析。当需要用 MCP 工具做流量分析/协议逆向/接口提取时使用。
---

# MCP 分析工作流

## 常见场景

| 场景 | 操作 |
|------|------|
| 游戏协议分析 | MCP 抓包 -> 解析字段 -> 组包自动化 |
| 下单流程分析 | MCP 抓包 -> 分析接口 -> 提取验证参数 |
| JS 逆向 | jshook MCP -> 分析加密函数 |
| 进程分析 | CE/x64dbg MCP -> 查找目标地址 |

## 典型流程

1. 启动 MCP 连接目标
2. 抓取网络请求或内存数据
3. 分析关键接口参数
4. 提取算法/生成自动化代码

## MCP 管理

- MCP 自建目录: `E:\mcp`
- 配置: 项目级 `.cursor/mcp.json` 或全局配置
- 安装新 MCP: 添加配置 -> 测试连接

## 游戏协议多目标配置

- 多游戏共用同一 TCP/抓包工具时：用 `game_profiles.json` 存多套 profile（id、name、header_size、aes_key、aes_iv、msg_names），运行时 `current_game_id` 持久化在 config；crypto/解析/组包均按 `get_current_profile()` 取当前游戏，GUI 提供游戏切换并刷新标题。
- 新游戏：追加 profile，必要时补 msg_names 表；发布时把 game_profiles.json 一并打进发布目录。

## Frida 桥接 MCP（无 CDP 时）

- 目标进程无对外 CDP 端口时：Frida 脚本在进程内抓 Network/Console 或执行逻辑，经 `send()` 发到 Node；Node 开本地 HTTP（如 29222）暴露 `/network`、`/console`、`POST /evaluate`、`GET /dom`，MCP 连该 HTTP 即获得类 DevTools 能力。
- 反向调用：MCP 的 evaluate 请求由 Node 经 `script.postMessage()` 发 Frida，脚本在能访问页面上下文时执行后 `send()` 回 Node 再响应 HTTP。主进程拿不到页面上下文时可先做 Phase1（仅 Network/通道），Phase2 再在渲染进程 attach 或注入页面 JS 与桥接通信。

## SunnyNet 配套 HTTP API 抓取 (IM/客服系统)

IM 系统的 WebSocket 协议之外，通常有配套的 HTTP 管理接口。用 SunnyNet 抓取的标准流程：

1. 在浏览器中操作目标功能（如转移客户、设置在线状态）
2. `request_search(url="关键词")` 按 URL 关键词筛选
3. `request_get(theology=id)` 查看完整请求体和响应体
4. HTTP API 通常无加密，Cookie 鉴权 + JSON body，直接 requests 调用

### 实例：快手客服 HTTP API

| 接口 | 方法 | 用途 |
|------|------|------|
| /cs/assistant/stat/colleagueView | GET | 获取可转移的在线同事列表 |
| /cs/assistant/session/batchTransferSessions | POST | 批量转移客户会话 |
| /cs/assistant/sessionpool/count | GET | 获取会话池排队数 |
| /cs/assistant/stat/personalView | GET | 个人接待统计 |

## SunnyNet 加密参数对照分析

当目标使用加密提交（如验证码 fverify），可通过 SunnyNet 抓多组真实请求来反推加密逻辑：

1. **抓 3+ 组相同操作的请求**: `request_search(url="fverify")` 或按关键词搜索
2. **提取加密参数**: `request_get(theology=id)` 获取完整URL参数
3. **跨会话对比**: 相同加密值 = 固定key + 固定明文；不同 = 动态参数
4. **配合 JS 分析**: 从 SDK 提取候选 key，结合已知明文攻击反推（详见 `shumei-captcha-reverse` skill）

此方法适用于 DES/AES-ECB 等确定性加密（相同输入相同输出），不适用于 CBC/CTR 等带 IV 的模式。

## 注意事项

- 分析结果输出完整 JSON 响应（HTTP 状态码、Content-Type）
- 数据自动保存到文件
- 大型分析任务用多 agent 并行
- 调试日志和注释保留（用户偏好）
