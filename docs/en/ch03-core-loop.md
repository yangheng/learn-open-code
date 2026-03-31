# Chapter 3: The Core Loop — Message, LLM, Stop Reason

> **Motto**: One while(true) powers the entire AI coding assistant.

## Where We Left Off

Agent and Model resolved. `SessionPrompt.loop()` enters its main while loop.

## Code Path

### The Loop Structure

```typescript
// src/session/prompt.ts (loop)
while (true) {
  const msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))
  // Find last user/assistant messages
  const model = await Provider.getModel(lastUser.model.providerID, lastUser.model.modelID)
  
  // Create empty assistant message (so TUI shows "typing...")
  const processor = await SessionProcessor.create({ assistantMessage, sessionID, model, abort })
  
  // Call LLM
  const result = await processor.process({ user, agent, system, messages, tools, model })
  
  if (result === "stop") break
  if (result === "compact") { await SessionCompaction.create(...); continue }
  continue  // "continue" = tool calls done, loop back for next LLM call
}
```

### Stream Processing

```typescript
// src/session/processor.ts:L184
const stream = llm.stream(streamInput)
yield* stream.pipe(Stream.tap((event) => handleEvent(event)), Stream.runDrain)
```

Key events: `text-delta` (append text), `tool-call` (execute tool), `finish-step` (check overflow).

### Three Exit Signals

- **`"stop"`**: LLM finished, error, or user aborted
- **`"compact"`**: context overflow, needs compaction
- **`"continue"`**: tool calls completed, loop back for more

## Diagram

![ch03-core-loop](../images/ch03-core-loop.webp)

## Key Insights

1. **Loop is in prompt.ts, not processor.ts** — Processor handles one LLM call, loop decides continuation
2. **Database is Single Source of Truth** — re-read messages each iteration
3. **Manual loop management** — OpenCode doesn't use AI SDK's auto-tool-execution

## Next: How tools are dispatched → [Chapter 4](./ch04-tools.md)

---

← [上一章：Chapter 2: Routing](./ch02-routing.md) | [下一章：Chapter 4: Tool Dispatch](./ch04-tools.md) →
