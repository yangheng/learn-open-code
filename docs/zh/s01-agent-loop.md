# S01 — Agent 循环：消息进，工具出，循环不止

> **格言：** "消息进，工具出，循环不止"

[下一课 → S02 工具系统](s02-tool-system.md)

---

## 问题

一个 AI 编程 Agent 的核心是什么？是一个**循环**：接收用户消息 → 调用 LLM → LLM 返回工具调用 → 执行工具 → 将结果送回 LLM → 重复直到 LLM 不再调用工具。

OpenCode 如何实现这个循环？

## 架构概览

```
用户消息
   │
   ▼
┌──────────────────────────────────┐
│         Session.chat()           │
│  创建 user message + assistant   │
│  message，触发 processor         │
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
│  │    ├─ text-delta → 追加文本 │ │
│  │    ├─ tool-call → 执行工具  │ │
│  │    ├─ tool-result → 记录    │ │
│  │    └─ finish → 检查是否继续 │ │
│  └─────────────────────────────┘ │
│                                  │
│  返回: "continue" | "stop" |     │
│        "compact"                 │
└──────────────────────────────────┘
```

## 核心代码

### Agent 定义

Agent 不是循环本身，而是**配置**。OpenCode 预定义了多个 Agent，每个有不同的权限和用途：

```typescript
// packages/opencode/src/agent/agent.ts

export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),
    native: z.boolean().optional(),
    hidden: z.boolean().optional(),
    permission: Permission.Ruleset,
    model: z.object({
      modelID: ModelID.zod,
      providerID: ProviderID.zod,
    }).optional(),
    prompt: z.string().optional(),
    options: z.record(z.string(), z.any()),
    steps: z.number().int().positive().optional(),
  })
}
```

内置 Agent 包括：
- **build** — 默认 Agent，拥有完整工具权限
- **plan** — 规划模式，禁止编辑工具
- **explore** — 只读探索，仅允许搜索/读取
- **general** — 通用子 Agent
- **compaction** — 上下文压缩（隐藏）
- **title** / **summary** — 自动标题和摘要（隐藏）

### SessionProcessor — 真正的循环

循环的核心在 `session/processor.ts`。`SessionProcessor.create()` 返回一个 `Handle`，其中的 `process()` 方法执行一轮 LLM 调用：

```typescript
// packages/opencode/src/session/processor.ts

export namespace SessionProcessor {
  export const DOOM_LOOP_THRESHOLD = 3

  export type Result = "compact" | "stop" | "continue"

  export interface Handle {
    readonly message: MessageV2.Assistant
    readonly partFromToolCall: (toolCallID: string) => MessageV2.ToolPart | undefined
    readonly abort: () => Effect.Effect<void>
    readonly process: (streamInput: LLM.StreamInput) => Effect.Effect<Result>
  }

  interface ProcessorContext extends Input {
    toolcalls: Record<string, MessageV2.ToolPart>
    shouldBreak: boolean
    snapshot: string | undefined
    blocked: boolean
    needsCompaction: boolean
    currentText: MessageV2.TextPart | undefined
    reasoningMap: Record<string, MessageV2.ReasoningPart>
  }
}
```

### 流式事件处理

`process()` 内部消费 LLM 流的每个事件：

```typescript
// packages/opencode/src/session/processor.ts (handleEvent)

case "tool-call": {
  const match = ctx.toolcalls[value.toolCallId]
  if (!match) return
  // 更新工具状态为 running
  ctx.toolcalls[value.toolCallId] = yield* session.updatePart({
    ...match,
    tool: value.toolName,
    state: { status: "running", input: value.input, time: { start: Date.now() } },
  })
  // Doom loop 检测 —— 同一工具+同一参数连续调用 3 次
  const parts = yield* Effect.promise(() => MessageV2.parts(ctx.assistantMessage.id))
  const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)
  // ...如果检测到 doom loop，请求权限中断
}
```

### LLM 流式调用

`session/llm.ts` 封装了对 Vercel AI SDK `streamText()` 的调用：

```typescript
// packages/opencode/src/session/llm.ts

export async function stream(input: StreamInput) {
  const [language, cfg, provider, auth] = await Promise.all([
    Provider.getLanguage(input.model),
    Config.get(),
    Provider.getProvider(input.model.providerID),
    Auth.get(input.model.providerID),
  ])

  // 组装 system prompt
  const system: string[] = []
  system.push([
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ].filter(x => x).join("\n"))

  // 调用 streamText
  return streamText({
    temperature: params.temperature,
    tools,
    messages,
    model: wrapLanguageModel({ /* ... */ }),
    // ...
  })
}
```

### 工具修复机制

OpenCode 内置了工具调用修复：如果 LLM 返回了错误的工具名，会尝试修复：

```typescript
// packages/opencode/src/session/llm.ts

async experimental_repairToolCall(failed) {
  const lower = failed.toolCall.toolName.toLowerCase()
  if (lower !== failed.toolCall.toolName && tools[lower]) {
    return { ...failed.toolCall, toolName: lower }
  }
  // 无法修复 → 转发到 "invalid" 工具
  return {
    ...failed.toolCall,
    input: JSON.stringify({
      tool: failed.toolCall.toolName,
      error: failed.error.message,
    }),
    toolName: "invalid",
  }
}
```

## 关键设计模式

### 1. Effect-TS 依赖注入

整个系统基于 Effect-TS 的 `ServiceMap` + `Layer` 模式：

```typescript
export class Service extends ServiceMap.Service<Service, Interface>()("@opencode/Agent") {}

export const layer = Layer.effect(Service, Effect.gen(function* () {
  const config = yield* Config.Service
  // ...
  return Service.of({ get, list, defaultAgent, generate })
}))
```

这意味着每个模块都可以声明依赖、被替换、被测试。

### 2. InstanceState — 项目级状态

`InstanceState` 是一个关键抽象，它让状态与当前项目实例绑定：

```typescript
const state = yield* InstanceState.make<State>(
  Effect.fn("Agent.state")(function* (ctx) {
    // ctx 包含 directory, worktree, project 等
    // 返回的状态只在当前实例有效
  })
)
```

### 3. 三态返回

`process()` 返回三种状态：
- `"continue"` → 有工具调用，需要再次循环
- `"stop"` → LLM 完成，无工具调用
- `"compact"` → 上下文太长，需要压缩后重试

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 循环实现 | `SessionProcessor` + Effect-TS | 内部循环（闭源） |
| Agent 配置 | 声明式 (Zod schema) | 固定模式 |
| 多 Agent | 内置 build/plan/explore/general | 单一 Agent |
| Doom loop 检测 | 显式阈值检查 | 未知 |
| 流式处理 | Vercel AI SDK `streamText()` | 自有实现 |

---

[下一课 → S02 工具系统](s02-tool-system.md)
