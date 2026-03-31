# 第一章：入口 —— 从 CLI 到 Session

> **格言**：每一次对话，都从一个会话开始。

## 起点

用户在终端敲下命令：

```bash
opencode run "fix the bug in auth.ts"
```

这条消息如何穿越整个系统？让我们从最开始跟踪。

## 代码路径

### 1. CLI 入口：RunCommand

OpenCode 使用 `yargs` 处理命令行。`run` 命令定义在：

```typescript
// src/cli/cmd/run.ts:L308
export const RunCommand = cmd({
  command: "run [message..]",
  describe: "run opencode with a message",
  handler: async (args) => {
    // ...
    await bootstrap(process.cwd(), async () => {
      const fetchFn = (async (input: RequestInfo | URL, init?: RequestInit) => {
        const request = new Request(input, init)
        return Server.Default().fetch(request)
      }) as typeof globalThis.fetch
      const sdk = createOpencodeClient({ baseUrl: "http://opencode.internal", fetch: fetchFn })
      await execute(sdk)
    })
  },
})
```

关键洞察：**`run` 命令并不直接调用核心逻辑**。它启动一个内嵌的 HTTP 服务器（`Server.Default()`），然后通过 SDK 客户端与之通信。这意味着 CLI 和 TUI 走的是**完全相同的 API 路径**。

### 2. 创建 Session

`execute()` 中的第一步是创建（或复用）一个 Session：

```typescript
// src/cli/cmd/run.ts（execute 函数内部）
const sessionID = await session(sdk)

async function session(sdk: OpencodeClient) {
  const result = await sdk.session.create({ title: name, permission: rules })
  return result.data?.id
}
```

SDK 调用到达服务端后，实际执行的是：

```typescript
// src/session/index.ts:L162
const createNext = Effect.fn("Session.createNext")(function* (input) {
  const result: Info = {
    id: SessionID.descending(input.id),
    slug: Slug.create(),
    version: Installation.VERSION,
    projectID: Instance.project.id,
    directory: input.directory,
    title: input.title ?? createDefaultTitle(),
    time: {
      created: Date.now(),
      updated: Date.now(),
    },
  }
  yield* Effect.sync(() => SyncEvent.run(Event.Created, { sessionID: result.id, info: result }))
  return result
})
```

Session 本质上是一行数据库记录，存在 SQLite 的 `session` 表中。它包含：
- **id**：使用 descending ULID，最新的排最前
- **slug**：人类可读的短标识
- **projectID**：关联到当前项目
- **directory**：工作目录

### 3. 发送用户消息

Session 创建后，CLI 发送用户的 prompt：

```typescript
// src/cli/cmd/run.ts（execute 函数内部）
await sdk.session.prompt({
  sessionID,
  parts: [{ type: "text", text: message }],
})
```

这触发了 `SessionPrompt.prompt()`，它做两件事：
1. 创建用户消息（`createUserMessage`）
2. 启动主循环（`loop()`）

```typescript
// src/session/prompt.ts:L84
export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID)
  const message = await createUserMessage(input)
  await Session.touch(input.sessionID)
  return loop({ sessionID: input.sessionID })
})
```

### 4. 用户消息的构造

`createUserMessage` 将用户输入转换为结构化消息：

```typescript
// src/session/prompt.ts（createUserMessage 内部）
const info: MessageV2.Info = {
  id: MessageID.ascending(),
  role: "user",
  sessionID: input.sessionID,
  agent: agent.name,        // 默认 "build"
  model: model,              // 如 "anthropic/claude-sonnet-4"
  time: { created: Date.now() },
}
```

注意：用户消息不仅包含文本，还绑定了 **agent 名称** 和 **model 信息**。这些将在下一章决定如何路由请求。

## 架构图

![ch01-entry-point](../images/ch01-entry-point.webp)

## 关键洞察

1. **CLI 是 Server 的客户端**：`opencode run` 不直接调用业务逻辑，而是通过内嵌 HTTP 服务器的 fetch 接口走 API
2. **Session 是一切的锚点**：所有消息、工具调用、权限都挂在 Session 上
3. **用户消息携带路由信息**：`agent` 和 `model` 字段决定了后续的处理链路

## 下一章预告

用户消息已经创建并存入数据库，`loop()` 已经被调用。但 LLM 是哪个？Agent 有什么权限？下一章我们跟进 Agent 和 Provider 的选择过程。
