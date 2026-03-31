# S11 — Snapshot & Worktree: Edit Boldly, Revert Freely

> **Motto:** "Edit boldly, revert freely"

[← S10 Plugin System](s10-plugin-system.md) | [S12 Client/Server →](s12-client-server.md)

---

**Snapshot** uses a separate Git repository (not the project's `.git`) at `$DATA/snapshot/$projectID/$hash/` to track file changes before each tool call. Operations: `track()` (commit snapshot), `restore()` (checkout), `revert()` (apply reverse patch), `diff()`/`diffFull()`. Files over 2MB are skipped. Snapshots older than 7 days are pruned. Semaphore-based locking prevents concurrent git operation conflicts.

**Worktree** uses `git worktree add` to create isolated working directories for parallel agent execution. Each worktree gets its own branch. Events: `Ready`, `Failed`. Supports custom start commands.

See [Chinese version](../zh/s11-snapshot-worktree.md) for full code analysis.

---

[← S10 Plugin System](s10-plugin-system.md) | [S12 Client/Server →](s12-client-server.md)
