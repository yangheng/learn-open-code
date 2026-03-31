# Chapter 10: Architecture — Worktree & Client/Server

> **Motto**: Good architecture makes complexity a simple composition.

## Where We Left Off

Snapshots make every change trackable and reversible. Now: parallel tasks and why it's a server.

## Worktree: Parallel Isolation

```typescript
// src/worktree/index.ts
// Uses Git's native worktree feature
git worktree add --no-checkout -b opencode/fix-auth ~/.opencode/data/worktree/<project>/<name>
git reset --hard  // Populate files
// Then: bootstrap project instance, run start scripts
```

Full filesystem isolation on independent Git branches.

## Client/Server Architecture

Even `opencode run` talks to an embedded HTTP server:

```typescript
// Local: zero-network in-memory fetch
const fetchFn = async (input, init) => Server.Default().fetch(new Request(input, init))
// Remote: HTTP
const sdk = createOpencodeClient({ baseUrl: "http://server:4096" })
```

Multiple clients supported: **TUI**, **CLI**, **Web UI**, **remote attach**.

### Event Bus → SSE

All state changes flow through Bus → SSE → clients. No polling.

```typescript
Bus.publish(Event, properties) → PubSub → SSE stream → client
```

## Diagram

![ch10-architecture](../images/ch10-architecture.webp)

## Key Insights

1. **Worktree = Git worktree**: native Git feature for filesystem-level isolation
2. **Server-first**: even local CLI uses HTTP API, ensuring consistency
3. **SSE is the only event channel**: all state changes push through Bus → SSE
4. **Embedded fetch**: local mode is zero-latency in-memory calls

## Full Journey Summary

```
CLI input → Session → Agent + Model routing
→ Core loop (message → LLM → tool call → permission → result)
→ Auto-compaction on overflow → Layered system prompts
→ MCP tools + LSP intelligence → Snapshot every change
→ Worktree parallel isolation → Client/Server architecture
```

Each layer is independent, composed through clear interfaces. This is OpenCode's design philosophy: **each module does one thing, and does it well**.
