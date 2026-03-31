# S10 — Plugin System: Hook-Driven, Infinitely Extensible

> **Motto:** "Hook-driven, infinitely extensible"

[← S09 LSP Integration](s09-lsp-integration.md) | [S11 Snapshot & Worktree →](s11-snapshot-worktree.md)

---

Plugins use a hook-based architecture where each hook follows `(input, output) => Promise<void>` — output is mutable, enabling chain processing. Key hooks: `tool.definition` (modify tool descriptions), `chat.params` (adjust LLM parameters), `chat.headers` (inject headers), `experimental.chat.system.transform` (modify system prompts). Built-in auth plugins: Copilot, Codex, GitLab, Poe. User plugins are loaded from npm packages or local paths with compatibility checking.

See [Chinese version](../zh/s10-plugin-system.md) for full code analysis.

---

[← S09 LSP Integration](s09-lsp-integration.md) | [S11 Snapshot & Worktree →](s11-snapshot-worktree.md)
