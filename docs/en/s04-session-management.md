# S04 — Session Management: Conversations are Database Rows

> **Motto:** "Conversations are database rows"

[← S03 Provider System](s03-provider-system.md) | [S05 Permission System →](s05-permission-system.md)

---

Sessions are persisted in SQLite via Drizzle ORM. Each `Session.Info` has id, slug, projectID, parentID (for subagent chains), title, summary (additions/deletions/files), and timestamps. Messages use `MessageV2` format with typed parts: `TextPart`, `ReasoningPart`, `ToolPart` (with states: pending→running→completed/error), and `FilePart`. Sessions support forking and automatic context compaction when conversations get too long.

See [Chinese version](../zh/s04-session-management.md) for full code analysis.

---

[← S03 Provider System](s03-provider-system.md) | [S05 Permission System →](s05-permission-system.md)
