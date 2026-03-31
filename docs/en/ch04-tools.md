# Chapter 4: Tool Dispatch — bash, read, write, edit

> **Motto**: LLM is the brain, tools are the hands.

## Where We Left Off

LLM stream emits a `tool-call` event. How are tools registered and executed?

## Code Path

### Tool Registration

```typescript
// src/session/prompt.ts:L380 (resolveTools)
// 1. Built-in tools from ToolRegistry
for (const item of await ToolRegistry.tools(model, agent)) {
  tools[item.id] = tool({ description, inputSchema, execute: wrapWithPluginHooks(item.execute) })
}
// 2. MCP tools merged in
for (const [key, item] of Object.entries(await MCP.tools())) {
  tools[key] = item
}
```

### Tool.define() Pattern

Every tool follows: **validate → execute → truncate**

```typescript
// src/tool/tool.ts:L38
Tool.define("bash", async () => ({
  parameters: z.object({ command: z.string() }),
  async execute(args, ctx) {
    await ctx.ask({ permission: "bash", patterns: [args.command] })  // Permission check
    const result = await Shell.execute(args.command)
    return { title: args.command, output: result.stdout, metadata: { exitCode } }
  },
}))
```

### Built-in Tools

~15 tools: `bash`, `read`, `write`, `edit`, `multiedit`, `glob`, `grep`, `list`, `task` (subagent), `skill`, `webfetch`, `websearch`, `codesearch`, `lsp`, `plan`.

### Tool Context

```typescript
ctx = {
  sessionID, messageID, agent, abort,
  messages,           // Full conversation history
  metadata(input),    // Update status in real-time
  ask(input),         // Request permission (next chapter)
}
```

## Diagram

![ch04-tools](../images/ch04-tools.webp)

## Key Insights

1. **Declarative tools**: `Tool.define()` provides schema + execute, framework handles validation/truncation
2. **Two sources**: built-in (ToolRegistry) + MCP, merged in resolveTools
3. **Tools don't know about LLM**: they receive args + context, return results
4. **Auto-truncation**: `Truncate.output()` prevents tools from blowing up context

## Next: Permission system → [Chapter 5](./ch05-permissions.md)

---

← [上一章：Chapter 3: The Core Loop](./ch03-core-loop.md) | [下一章：Chapter 5: Permissions](./ch05-permissions.md) →
