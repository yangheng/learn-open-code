# S02 — 工具系统：每个工具都是一个 `define()` 调用

> **格言：** "每个工具都是一个 `define()` 调用"

[← S01 Agent 循环](s01-agent-loop.md) | [S03 Provider 系统 →](s03-provider-system.md)

---

## 问题

AI Agent 的核心能力来自工具。Agent 本身只能生成文本，但工具让它能读文件、写代码、执行命令。OpenCode 如何设计一个统一的工具系统，既支持内置工具又允许自定义扩展？

## 架构概览

```
┌─────────────────────────────────────────────┐
│              ToolRegistry                    │
│                                             │
│  内置工具:                                   │
│  ┌────┐ ┌────┐ ┌────┐ ┌─────┐ ┌──────┐    │
│  │bash│ │read│ │edit│ │write│ │glob  │    │
│  └────┘ └────┘ └────┘ └─────┘ └──────┘    │
│  ┌────┐ ┌────┐ ┌────┐ ┌─────┐ ┌──────┐    │
│  │grep│ │task│ │todo│ │fetch│ │search│    │
│  └────┘ └────┘ └────┘ └─────┘ └──────┘    │
│                                             │
│  动态工具:                                   │
│  ┌──────────┐  ┌──────────┐                 │
│  │ Plugin   │  │  Custom  │                 │
│  │ Tools    │  │ .ts/.js  │                 │
│  └──────────┘  └──────────┘                 │
│                                             │
│  模型适配:                                   │
│  GPT → apply_patch 代替 edit/write          │
│  其他 → edit + write                         │
└─────────────────────────────────────────────┘
```

## 核心代码

### Tool.define() — 工具定义接口

每个工具通过 `Tool.define()` 创建，这是整个系统的基石：

```typescript
// packages/opencode/src/tool/tool.ts

export namespace Tool {
  export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
    id: string
    init: (ctx?: InitContext) => Promise<{
      description: string
      parameters: Parameters
      execute(
        args: z.infer<Parameters>,
        ctx: Context,
      ): Promise<{
        title: string
        metadata: M
        output: string
        attachments?: Omit<MessageV2.FilePart, "id" | "sessionID" | "messageID">[]
      }>
    }>
  }

  export function define<Parameters extends z.ZodType, Result extends Metadata>(
    id: string,
    init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
  ): Info<Parameters, Result> {
    return {
      id,
      init: async (initCtx) => {
        const toolInfo = init instanceof Function ? await init(initCtx) : init
        const execute = toolInfo.execute
        // 包装 execute：参数校验 + 输出截断
        toolInfo.execute = async (args, ctx) => {
          try {
            toolInfo.parameters.parse(args)
          } catch (error) {
            if (error instanceof z.ZodError && toolInfo.formatValidationError) {
              throw new Error(toolInfo.formatValidationError(error), { cause: error })
            }
            throw new Error(`The ${id} tool was called with invalid arguments: ${error}`)
          }
          const result = await execute(args, ctx)
          // 自动截断过长输出
          const truncated = await Truncate.output(result.output, {}, initCtx?.agent)
          return {
            ...result,
            output: truncated.content,
            metadata: { ...result.metadata, truncated: truncated.truncated },
          }
        }
        return toolInfo
      },
    }
  }
}
```

**关键设计：**
1. **延迟初始化** — `init()` 是异步的，工具可以根据 Agent 上下文动态调整描述
2. **自动参数校验** — 在执行前用 Zod 验证参数
3. **自动输出截断** — 防止超长输出撑爆上下文

### Tool.Context — 执行上下文

每个工具执行时都能访问丰富的上下文：

```typescript
// packages/opencode/src/tool/tool.ts

export type Context<M extends Metadata = Metadata> = {
  sessionID: SessionID
  messageID: MessageID
  agent: string
  abort: AbortSignal
  callID?: string
  messages: MessageV2.WithParts[]
  metadata(input: { title?: string; metadata?: M }): void
  ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Promise<void>
}
```

`ask()` 方法尤其重要 —— 工具通过它请求用户授权执行危险操作。

### ToolRegistry — 工具注册中心

```typescript
// packages/opencode/src/tool/registry.ts

export namespace ToolRegistry {
  const all = Effect.fn("ToolRegistry.all")(function* (custom: Tool.Info[]) {
    const cfg = yield* config.get()
    return [
      InvalidTool,
      ...(question ? [QuestionTool] : []),
      BashTool, ReadTool, GlobTool, GrepTool,
      EditTool, WriteTool, TaskTool,
      WebFetchTool, TodoWriteTool, WebSearchTool,
      CodeSearchTool, SkillTool, ApplyPatchTool,
      ...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [LspTool] : []),
      ...(cfg.experimental?.batch_tool === true ? [BatchTool] : []),
      ...custom,
    ]
  })
}
```

### 模型适配 — GPT 用 apply_patch

```typescript
// packages/opencode/src/tool/registry.ts

const filtered = allTools.filter((tool) => {
  const usePatch =
    model.modelID.includes("gpt-") &&
    !model.modelID.includes("oss") &&
    !model.modelID.includes("gpt-4")
  if (tool.id === "apply_patch") return usePatch
  if (tool.id === "edit" || tool.id === "write") return !usePatch
  return true
})
```

GPT-5 等新模型使用 `apply_patch` 工具（统一格式的 diff），而其他模型使用 `edit` + `write`。

### TaskTool — 子 Agent 调度

`task` 工具是 OpenCode 实现多 Agent 协作的方式：

```typescript
// packages/opencode/src/tool/task.ts

const parameters = z.object({
  description: z.string().describe("A short (3-5 words) description of the task"),
  prompt: z.string().describe("The task for the agent to perform"),
  subagent_type: z.string().describe("The type of specialized agent to use"),
  task_id: z.string().optional(),  // 可恢复之前的子会话
})

export const TaskTool = Tool.define("task", async (ctx) => {
  const agents = await Agent.list().then(x => x.filter(a => a.mode !== "primary"))
  // ...
  return {
    description: DESCRIPTION.replace("{agents}", agentList),
    parameters,
    async execute(params, ctx) {
      // 创建子会话 (或恢复已有的)
      const session = params.task_id
        ? await Session.get(SessionID.make(params.task_id))
        : await Session.create({ parentID: ctx.sessionID, title: params.description })
      // 在子会话中执行
      const result = await Session.chat({ sessionID: session.id, ... })
      return { title: params.description, output: result, metadata: {} }
    }
  }
})
```

### 自定义工具加载

用户可以在 `.opencode/tool/` 目录放置自定义工具：

```typescript
// packages/opencode/src/tool/registry.ts

const dirs = yield* config.directories()
const matches = dirs.flatMap((dir) =>
  Glob.scanSync("{tool,tools}/*.{js,ts}", { cwd: dir, absolute: true })
)
for (const match of matches) {
  const namespace = path.basename(match, path.extname(match))
  const mod = await import(match)
  for (const [id, def] of Object.entries(mod)) {
    custom.push(fromPlugin(id === "default" ? namespace : `${namespace}_${id}`, def))
  }
}
```

## 内置工具一览

| 工具 | 功能 | 权限类别 |
|------|------|----------|
| `bash` | 执行 shell 命令 | `bash` |
| `read` | 读取文件内容 | `read` |
| `edit` | 编辑文件（搜索/替换） | `edit` |
| `write` | 创建/覆盖文件 | `edit` |
| `apply_patch` | 统一 diff 格式编辑 | `edit` |
| `glob` | 文件名模式匹配 | `glob` |
| `grep` | 文本搜索 | `grep` |
| `task` | 启动子 Agent | `task` |
| `todowrite` | 任务管理 | `todowrite` |
| `webfetch` | 获取 URL 内容 | `webfetch` |
| `websearch` | 网络搜索 | `websearch` |
| `codesearch` | 代码语义搜索 | `codesearch` |
| `skill` | 加载技能 | `skill` |
| `question` | 向用户提问 | `question` |
| `lsp` | LSP 操作 | `lsp` |
| `batch` | 批量工具调用 | 各工具权限 |

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 工具定义 | `Tool.define()` 统一接口 | 内置固定 |
| 参数校验 | Zod schema 自动验证 | 内部验证 |
| 输出截断 | 自动 `Truncate.output()` | 手动处理 |
| 自定义工具 | `.opencode/tool/*.ts` + Plugin | 无 |
| 子 Agent | `task` 工具 → 子会话 | 无原生支持 |
| 模型适配 | GPT→apply_patch, 其他→edit | 固定工具集 |

---

[← S01 Agent 循环](s01-agent-loop.md) | [S03 Provider 系统 →](s03-provider-system.md)
