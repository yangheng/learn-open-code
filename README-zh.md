[English](./README.md) | [中文](./README-zh.md)

# 📖 深入 OpenCode 源码：跟随一条请求从头到尾

> 这不是 API 文档，这是一趟旅程。我们跟踪用户输入的 `"fix the bug in auth.ts"` 如何穿越整个 OpenCode 系统。

## 架构总览

![overview](./docs/images/overview.webp)

```
用户输入 "fix the bug in auth.ts"
  → ch01: CLI 接收输入，创建 Session
  → ch02: Agent + Provider 选择，模型路由
  → ch03: 核心循环：消息 → LLM → stop_reason → 工具/文本
  → ch04: LLM 说 tool_use，调度 bash/read/write/edit
  → ch05: bash 执行前，用户必须批准
  → ch06: 上下文太长，自动压缩
  → ch07: 系统提示分层组装
  → ch08: MCP 外部工具 + LSP 代码智能
  → ch09: 出错了，用 Snapshot 撤销
  → ch10: 为什么是 Server，不只是 CLI
```

## 章节导航

| 章节 | 主题 | 核心问题 |
|------|------|----------|
| [第一章](./docs/zh/ch01-entry-point.md) | 入口：CLI → Session | 用户输入如何变成一个 Session？ |
| [第二章](./docs/zh/ch02-routing.md) | 路由：Agent + Provider | 谁来处理？用哪个模型？ |
| [第三章](./docs/zh/ch03-core-loop.md) | 核心循环 | while(true) 里到底在做什么？ |
| [第四章](./docs/zh/ch04-tools.md) | 工具调度 | LLM 说 tool_use 后发生了什么？ |
| [第五章](./docs/zh/ch05-permissions.md) | 权限系统 | 如何确保安全？用户如何批准？ |
| [第六章](./docs/zh/ch06-compaction.md) | 上下文管理 | 对话太长怎么办？ |
| [第七章](./docs/zh/ch07-prompts-skills.md) | 提示词 + Skill | 系统提示如何组装？ |
| [第八章](./docs/zh/ch08-mcp-lsp.md) | MCP + LSP | 如何扩展工具和代码智能？ |
| [第九章](./docs/zh/ch09-snapshots.md) | 快照与撤销 | 改错了怎么办？ |
| [第十章](./docs/zh/ch10-architecture.md) | 架构：Worktree + C/S | 为什么是这样的架构？ |

## 阅读建议

- **按顺序读**：每一章都承接上一章的结尾
- **对照源码**：每个代码片段都标注了文件路径
- **关注 "关键洞察"**：每章末尾的总结是精华

## 源码版本

基于 [OpenCode](https://github.com/nicepkg/opencode) 源码分析。

## License

MIT
