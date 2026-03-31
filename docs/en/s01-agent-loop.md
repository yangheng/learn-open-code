# S01 — Agent Loop: Message In, Tool Out, Loop Forever

> **Motto:** "Message in, tool out, loop forever"

[Next → S02 Tool System](s02-tool-system.md)

---

## The Problem

The core of an AI coding agent is a **loop**: receive user message → call LLM → LLM returns tool calls → execute tools → send results back → repeat until LLM stops calling tools.

## Architecture

```
User Message
   │
   ▼
┌──────────────────────────────────┐
│         Session.chat()           │
│  Creates user + assistant msgs   │
│  Triggers processor              │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│     SessionProcessor.create()    │
│  ┌─────────────────────────────┐ │
│  │  LLM.stream(streamInput)   │ │
│  │  ┌───────────────────────┐  │ │
│  │  │ streamText() (AI SDK) │  │ │
│  │  └───────────┬───────────┘  │ │
│  │              │              │ │
│  │  for await (event of stream)│ │
│  │    ├─ text-delta → append   │ │
│  │    ├─ tool-call → execute   │ │
│  │    ├─ tool-result → record  │ │
│  │    └─ finish → check loop   │ │
│  └─────────────────────────────┘ │
│                                  │
│  Returns: "continue"|"stop"|     │
│           "compact"              │
└──────────────────────────────────┘
```

## Key Code

### Agent as Configuration

Agents are not the loop itself — they are **configuration** objects defining permissions and behavior:

```typescript
// packages/opencode/src/agent/agent.ts
export const Info = z.object({
  name: z.string(),
  mode: z.enum(["subagent", "primary", "all"]),
  permission: Permission.Ruleset,
  model: z.object({ modelID: ModelID.zod, providerID: ProviderID.zod }).optional(),
  prompt: z.string().optional(),
  steps: z.number().int().positive().optional(),
})
```

Built-in agents: **build** (default), **plan** (read-only planning), **explore** (search only), **general** (subagent), **compaction** (context compression).

### SessionProcessor — The Real Loop

```typescript
// packages/opencode/src/session/processor.ts
export type Result = "compact" | "stop" | "continue"

export interface Handle {
  readonly message: MessageV2.Assistant
  readonly process: (streamInput: LLM.StreamInput) => Effect.Effect<Result>
}
```

`process()` returns:
- `"continue"` → tool calls present, loop again
- `"stop"` → LLM finished, no tool calls
- `"compact"` → context too long, compress and retry

### Doom Loop Detection

```typescript
// Same tool + same args called 3 times in a row → ask permission to break
const DOOM_LOOP_THRESHOLD = 3
```

### Tool Call Repair

```typescript
// packages/opencode/src/session/llm.ts
async experimental_repairToolCall(failed) {
  const lower = failed.toolCall.toolName.toLowerCase()
  if (lower !== failed.toolCall.toolName && tools[lower]) {
    return { ...failed.toolCall, toolName: lower }
  }
  return { ...failed.toolCall, toolName: "invalid", input: JSON.stringify({ error: failed.error.message }) }
}
```

## Key Design Patterns

1. **Effect-TS DI** — `ServiceMap.Service` + `Layer` for dependency injection
2. **InstanceState** — State scoped to current project instance
3. **Three-state return** — continue/stop/compact drives the outer loop

---

[Next → S02 Tool System](s02-tool-system.md)
