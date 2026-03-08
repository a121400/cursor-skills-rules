---
name: mcp-analysis-workflow
description: MCP 工具分析工作流。流量分析、协议逆向、接口提取的通用模式。
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

## 注意事项

- 分析结果输出完整 JSON 响应（HTTP 状态码、Content-Type）
- 数据自动保存到文件
- 大型分析任务用多 agent 并行
- 调试日志和注释保留（用户偏好）
