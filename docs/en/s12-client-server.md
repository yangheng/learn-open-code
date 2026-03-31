# S12 — Client/Server Architecture: CLI is Just One Client

> **Motto:** "CLI is just one client"

[← S11 Snapshot & Worktree](s11-snapshot-worktree.md)

---

OpenCode runs as an HTTP server (Hono framework) serving CLI, Desktop (Tauri), and Web clients. Features: CORS for localhost/Tauri/opencode.ai origins, optional BasicAuth, compression (skipping SSE endpoints), and workspace routing for multi-project support. The **Bus** event system (Effect-TS `PubSub`) propagates events globally — session updates, permission requests, MCP tool changes, etc. SSE endpoint `/event` streams all events to clients. `BusEvent.define()` creates typed events with Zod schemas, auto-generating OpenAPI specs via `hono-openapi`. mDNS enables desktop app auto-discovery.

See [Chinese version](../zh/s12-client-server.md) for full code analysis.

---

[← S11 Snapshot & Worktree](s11-snapshot-worktree.md)
