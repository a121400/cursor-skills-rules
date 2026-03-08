---
name: everything-claude-code-agents
description: Everything Claude Code 专用代理集合。根据任务类型选择合适的 agent 指南进行工作。
---

# Everything Claude Code Agents

本目录包含从 everything-claude-code 迁移的专家代理指南。在相应场景下读取对应 .md 文件作为参考。

## 可用 Agents

| Agent | 文件 | 使用场景 |
|-------|------|----------|
| planner | planner.md | 复杂功能实现、重构规划 |
| architect | architect.md | 系统架构设计决策 |
| tdd-guide | tdd-guide.md | 测试驱动开发 |
| code-reviewer | code-reviewer.md | 代码质量与安全审查 |
| security-reviewer | security-reviewer.md | 漏洞分析 |
| build-error-resolver | build-error-resolver.md | 构建错误修复 |
| e2e-runner | e2e-runner.md | Playwright E2E 测试 |
| refactor-cleaner | refactor-cleaner.md | 死代码清理 |
| doc-updater | doc-updater.md | 文档同步 |

## 何时使用

- 复杂功能请求 -> 使用 planner
- 代码刚写/修改完 -> 使用 code-reviewer
- 新功能或 Bug 修复 -> 使用 tdd-guide
- 架构决策 -> 使用 architect
- 构建失败 -> 使用 build-error-resolver
