# S02 — Tool System: Every Tool is a `define()` Call

> **Motto:** "Every tool is a `define()` call"

[← S01 Agent Loop](s01-agent-loop.md) | [S03 Provider System →](s03-provider-system.md)

---

## Key Code

### Tool.define() — The Universal Interface

```typescript
// packages/opencode/src/tool/tool.ts
export function define<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
): Info<Parameters, Result>
```

Every tool returns `{ title, metadata, output, attachments? }`. The wrapper automatically validates parameters with Zod and truncates oversized output via `Truncate.output()`.

### Tool Context

```typescript
export type Context = {
  sessionID: SessionID
  messageID: MessageID
  agent: string
  abort: AbortSignal
  messages: MessageV2.WithParts[]
  metadata(input: { title?: string; metadata?: any }): void
  ask(input: Permission.Request): Promise<void>  // Request user permission
}
```

### ToolRegistry — Dynamic Assembly

```typescript
// packages/opencode/src/tool/registry.ts
// Built-in: bash, read, edit, write, glob, grep, task, webfetch, todowrite, websearch, codesearch, skill, apply_patch
// Plus: custom tools from .opencode/tool/*.ts and plugins
```

### Model-Specific Tool Selection

```typescript
// GPT models → apply_patch (unified diff format)
// Other models → edit + write (search/replace)
const usePatch = model.modelID.includes("gpt-") && !model.modelID.includes("gpt-4")
```

### TaskTool — Subagent Dispatch

The `task` tool creates child sessions for parallel work with specialized agents (explore, general).

---

[← S01 Agent Loop](s01-agent-loop.md) | [S03 Provider System →](s03-provider-system.md)
