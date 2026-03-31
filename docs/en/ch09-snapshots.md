# Chapter 9: Snapshots & Undo

> **Motto**: The courage to modify comes from the ability to undo.

## Where We Left Off

MCP and LSP extend capabilities. But tools can make mistakes. Snapshots are the safety net.

## Code Path

### Shadow Git Repository

Snapshots use a **shadow Git repo** at `~/.opencode/data/snapshot/<project-hash>/`.

```
start-step  → git add -A && git commit → snapshot hash
              (tools execute, files change)
finish-step → git add -A && git commit → git diff old..new → patch
```

### Undo

```typescript
// src/snapshot/index.ts
const restore = (hash) => git(["checkout", hash, "--", "."])  // Restore to snapshot state
const revert = (patches) => patches.reverse().forEach(p => restore(p.hash))
```

### Session Diff

Snapshot also provides full session diff: which files changed, how many lines added/deleted.

## Diagram

![ch09-snapshots](../images/ch09-snapshots.webp)

## Key Insights

1. **Git as snapshot engine**: leverages Git's incremental storage
2. **Step granularity**: one snapshot per LLM step (may contain multiple tool calls)
3. **Patch records changes**: not just undo, but also "what changed"
4. **Shadow repo isolation**: doesn't pollute project Git history

## Next: Architecture → [Chapter 10](./ch10-architecture.md)
