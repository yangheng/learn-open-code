# S05 — Permission System: Wildcard Rules, Layered Merge

> **Motto:** "Wildcard rules, layered merge"

[← S04 Session Management](s04-session-management.md) | [S06 Prompt Engineering →](s06-prompt-engineering.md)

---

Each permission rule is `{ permission, pattern, action }` where action is `allow|deny|ask`. Rules use wildcard matching and are merged in layers: defaults → agent-specific → user config → session approved. At runtime, `Permission.ask()` uses Effect-TS `Deferred` to block until the user replies `once|always|reject`. Three error types: `DeniedError` (rule blocks it), `RejectedError` (user said no), `CorrectedError` (user said no with feedback).

See [Chinese version](../zh/s05-permission-system.md) for full code analysis.

---

[← S04 Session Management](s04-session-management.md) | [S06 Prompt Engineering →](s06-prompt-engineering.md)
