# S08 — MCP Integration: Protocol is Interop

> **Motto:** "Protocol is interop"

[← S07 Skill System](s07-skill-system.md) | [S09 LSP Integration →](s09-lsp-integration.md)

---

OpenCode implements full MCP client support with three transports: stdio, SSE, and Streamable HTTP. MCP tools are converted to AI SDK `Tool` type via `convertMcpTool()`. OAuth authentication is supported with browser-based flows. Tool list changes are propagated via `Bus` events. MCP prompts are registered as slash commands. Default timeout is 30 seconds per tool call.

See [Chinese version](../zh/s08-mcp-integration.md) for full code analysis.

---

[← S07 Skill System](s07-skill-system.md) | [S09 LSP Integration →](s09-lsp-integration.md)
