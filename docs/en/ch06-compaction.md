# Chapter 6: Context Management — Compaction & Pruning

> **Motto**: Forgetting is the key to better memory.

## Where We Left Off

Each `finish-step` checks token usage. When approaching model context limits, compaction triggers.

## Code Path

### Two-level Strategy

**Level 1: Pruning** (cheap) — erase old tool outputs
```typescript
// src/session/compaction.ts:L41 (prune)
// Scan backwards, protect last 2 turns + 40K tokens of tool output
// Mark older tool outputs as compacted (omitted when building model messages)
if (total > PRUNE_PROTECT) toPrune.push(part)
if (pruned > PRUNE_MINIMUM) {
  for (const part of toPrune) part.state.time.compacted = Date.now()
}
```

**Level 2: Compaction** (expensive) — summarize entire conversation
```typescript
// src/session/compaction.ts:L75 (process)
// Use "compaction" agent (no tool access)
// Send all history + summary prompt to LLM
// Result becomes a summary message (summary=true)
// All messages before summary are filtered out in subsequent loops
```

### Auto-continue After Compaction

```typescript
// After compaction, inject synthetic "Continue..." message
yield* session.updatePart({
  type: "text", synthetic: true,
  text: "Continue if you have next steps, or stop and ask for clarification.",
})
```

## Diagram

![ch06-compaction](../images/ch06-compaction.webp)

## Key Insights

1. **Two levels**: prune old tool outputs first, then full compaction if needed
2. **Dedicated agent**: compaction uses a tool-less agent
3. **summary=true is the watermark**: everything before it gets filtered
4. **Transparent to user**: auto-compaction + synthetic continue message

## Next: System prompt assembly → [Chapter 7](./ch07-prompts-skills.md)
