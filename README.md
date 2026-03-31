# learn-open-code

Deep source-code analysis of **[OpenCode](https://github.com/anomalyco/opencode)** — the open-source AI coding agent CLI built in TypeScript.

Inspired by [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code), this project dissects OpenCode's architecture through 12 progressive sessions, from the agent loop to the client/server design.

> 📖 **[中文文档 (推荐)](README-zh.md)** — The Chinese version is the primary, most comprehensive documentation.

## Sessions

| # | Topic | Module | Motto |
|---|-------|--------|-------|
| [01](docs/en/s01-agent-loop.md) | Agent Loop | `agent/`, `session/processor.ts` | "Message in, tool out, loop forever" |
| [02](docs/en/s02-tool-system.md) | Tool System | `tool/` | "Every tool is a `define()` call" |
| [03](docs/en/s03-provider-system.md) | Provider System | `provider/` | "20+ providers, one `streamText()`" |
| [04](docs/en/s04-session-management.md) | Session Management | `session/` | "Conversations are database rows" |
| [05](docs/en/s05-permission-system.md) | Permission System | `permission/` | "Wildcard rules, layered merge" |
| [06](docs/en/s06-prompt-engineering.md) | Prompt Engineering | `session/prompt/`, `agent/prompt/` | "One persona per model" |
| [07](docs/en/s07-skill-system.md) | Skill System | `skill/` | "Markdown is capability" |
| [08](docs/en/s08-mcp-integration.md) | MCP Integration | `mcp/` | "Protocol is interop" |
| [09](docs/en/s09-lsp-integration.md) | LSP Integration | `lsp/` | "Editor smarts, terminal delivery" |
| [10](docs/en/s10-plugin-system.md) | Plugin System | `plugin/` | "Hook-driven, infinitely extensible" |
| [11](docs/en/s11-snapshot-worktree.md) | Snapshot & Worktree | `snapshot/`, `worktree/` | "Edit boldly, revert freely" |
| [12](docs/en/s12-client-server.md) | Client/Server | `server/`, `bus/` | "CLI is just one client" |

## Quick Start

```bash
git clone https://github.com/anomalyco/opencode.git
cd opencode
ls packages/opencode/src/
```

## License

MIT
