# S09 — LSP Integration: Editor Smarts, Terminal Delivery

> **Motto:** "Editor smarts, terminal delivery"

[← S08 MCP Integration](s08-mcp-integration.md) | [S10 Plugin System →](s10-plugin-system.md)

---

OpenCode integrates Language Server Protocol for code intelligence: diagnostics, hover, go-to-definition, find references, and document/workspace symbols. LSP servers (TypeScript, Pyright/ty, gopls) are spawned on demand. After file edits, `touchFile()` notifies the LSP server to re-analyze, creating a feedback loop for catching type errors. The LSP tool (experimental, behind `OPENCODE_EXPERIMENTAL_LSP_TOOL` flag) lets the agent directly query LSP operations.

See [Chinese version](../zh/s09-lsp-integration.md) for full code analysis.

---

[← S08 MCP Integration](s08-mcp-integration.md) | [S10 Plugin System →](s10-plugin-system.md)
