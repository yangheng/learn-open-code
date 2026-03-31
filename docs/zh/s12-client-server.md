# S12 — 客户端/服务端架构：CLI 只是客户端之一

> **格言：** "CLI 只是客户端之一"

[← S11 快照与工作树](s11-snapshot-worktree.md)

---

## 问题

AI Agent 不只是 CLI。OpenCode 支持 CLI、桌面应用、Web 界面。如何设计一个后端服务，让多种客户端共用同一个核心？

## 架构概览

```
┌──────────────────────────────────────────────────┐
│                   客户端                          │
│                                                  │
│  ┌──────┐    ┌──────────┐    ┌───────────┐       │
│  │ CLI  │    │ Desktop  │    │ Web (Hono) │       │
│  │ (TUI)│    │ (Tauri)  │    │           │       │
│  └──┬───┘    └────┬─────┘    └─────┬─────┘       │
│     │             │                │              │
│     ▼             ▼                ▼              │
│  ┌───────────────────────────────────────────┐    │
│  │         HTTP Server (Hono)                │    │
│  │  /session/*  /global/*  /event (SSE)      │    │
│  │                                           │    │
│  │  Middleware:                               │    │
│  │  • CORS (localhost, tauri://, opencode.ai)│    │
│  │  • BasicAuth (optional)                   │    │
│  │  • Compression                            │    │
│  │  • WorkspaceRouter (多项目路由)            │    │
│  └─────────────────────┬─────────────────────┘    │
│                        │                          │
│  ┌─────────────────────▼─────────────────────┐    │
│  │            Bus (事件总线)                   │    │
│  │  PubSub<Payload>                          │    │
│  │  • publish(event, properties)             │    │
│  │  • subscribe(event) → Stream              │    │
│  │  • subscribeAll() → Stream (SSE)          │    │
│  └───────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

## 核心代码

### Hono HTTP 服务

```typescript
// packages/opencode/src/server/server.ts

export const ControlPlaneRoutes = (opts?: { cors?: string[] }): Hono => {
  const app = new Hono()
  return app
    .onError(errorHandler(log))
    // 认证中间件
    .use((c, next) => {
      if (c.req.method === "OPTIONS") return next()  // CORS 预检放行
      const password = Flag.OPENCODE_SERVER_PASSWORD
      if (!password) return next()
      return basicAuth({ username: "opencode", password })(c, next)
    })
    // 日志中间件
    .use(async (c, next) => {
      log.info("request", { method: c.req.method, path: c.req.path })
      await next()
    })
    // CORS 配置
    .use(cors({
      maxAge: 86_400,
      origin(input) {
        if (input?.startsWith("http://localhost:")) return input
        if (input?.startsWith("http://127.0.0.1:")) return input
        // Tauri 桌面应用
        if (input === "tauri://localhost" ||
            input === "http://tauri.localhost") return input
        // opencode.ai 域名
        if (/^https:\/\/([a-z0-9-]+\.)*opencode\.ai$/.test(input)) return input
        return
      },
    }))
    // 压缩 (跳过 SSE 和大消息)
    .use((c, next) => {
      if (skipCompress(c.req.path, c.req.method)) return next()
      return zipped(c, next)
    })
    .route("/global", GlobalRoutes())
}
```

### Bus 事件总线

```typescript
// packages/opencode/src/bus/index.ts

export interface Interface {
  readonly publish: <D extends BusEvent.Definition>(
    def: D, properties: z.output<D["properties"]>,
  ) => Effect.Effect<void>
  readonly subscribe: <D extends BusEvent.Definition>(
    def: D,
  ) => Stream.Stream<Payload<D>>
  readonly subscribeAll: () => Stream.Stream<Payload>
  readonly subscribeCallback: <D extends BusEvent.Definition>(
    def: D, callback: (event: Payload<D>) => unknown,
  ) => Effect.Effect<() => void>
}
```

Bus 基于 Effect-TS 的 `PubSub`：

```typescript
// packages/opencode/src/bus/index.ts

const cache = yield* InstanceState.make<State>(
  Effect.fn("Bus.state")(function* (ctx) {
    const wildcard = yield* PubSub.unbounded<Payload>()
    const typed = new Map<string, PubSub.PubSub<Payload>>()

    // 实例销毁时发布 InstanceDisposed 事件
    yield* Effect.addFinalizer(() =>
      Effect.gen(function* () {
        yield* PubSub.publish(wildcard, {
          type: InstanceDisposed.type,
          properties: { directory: ctx.directory },
        })
        yield* PubSub.shutdown(wildcard)
        for (const ps of typed.values()) {
          yield* PubSub.shutdown(ps)
        }
      }),
    )
    return { wildcard, typed }
  }),
)
```

### BusEvent 类型注册

```typescript
// packages/opencode/src/bus/bus-event.ts

export namespace BusEvent {
  const registry = new Map<string, Definition>()

  export function define<Type extends string, Properties extends ZodType>(
    type: Type, properties: Properties,
  ) {
    const result = { type, properties }
    registry.set(type, result)
    return result
  }

  // 自动生成 discriminated union 用于 OpenAPI schema
  export function payloads() {
    return z.discriminatedUnion("type",
      registry.entries().map(([type, def]) =>
        z.object({
          type: z.literal(type),
          properties: def.properties,
        })
      ).toArray()
    )
  }
}
```

### SSE 事件流

服务端通过 `/event` 端点向客户端推送实时事件：

```
GET /event → Server-Sent Events
  data: {"type":"session.updated","properties":{...}}
  data: {"type":"permission.asked","properties":{...}}
  data: {"type":"message.updated","properties":{...}}
```

### 多项目路由

`WorkspaceRouterMiddleware` 让一个服务器同时服务多个项目：

```typescript
// packages/opencode/src/server/router.ts

// 根据请求路径或 header 路由到不同的项目实例
// 每个项目有独立的 Session、Agent、Permission 等状态
```

### MDNS 发现

```typescript
// packages/opencode/src/server/mdns.ts

// 使用 mDNS 让桌面应用自动发现本地运行的 OpenCode 服务
```

### OpenAPI 规范

```typescript
// packages/opencode/src/server/server.ts

import { describeRoute, generateSpecs, validator, resolver, openAPIRouteHandler } from "hono-openapi"

// 每个路由都有 OpenAPI 描述
// 可自动生成 API 文档和 SDK
```

## 事件系统遍布全局

整个 OpenCode 通过 Bus 事件通信：

```typescript
// 部分事件类型:
Session.Event.Updated    // 会话更新
Session.Event.Error      // 会话错误
Permission.Event.Asked   // 请求权限
Permission.Event.Replied // 权限回复
MCP.ToolsChanged         // MCP 工具变更
Worktree.Event.Ready     // 工作树就绪
LSP.Event.Updated        // LSP 状态更新
Command.Event.Executed   // 命令执行
Bus.InstanceDisposed     // 实例销毁
```

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 架构 | Client/Server (Hono) | 单进程 CLI |
| 客户端 | CLI + Desktop + Web | CLI only |
| 通信 | HTTP + SSE + WebSocket | N/A |
| 事件系统 | Effect-TS PubSub Bus | 无 |
| API 规范 | OpenAPI (hono-openapi) | 无 |
| 多项目 | WorkspaceRouter | 单目录 |
| 发现 | mDNS | 无 |

---

[← S11 快照与工作树](s11-snapshot-worktree.md)

---

## 🎉 恭喜完成！

你已经走完了 OpenCode 源码分析的全部 12 课。现在你应该对一个生产级 AI 编程 Agent 的架构有了深入的理解：

1. **Agent 循环** — 消息 → LLM → 工具 → 循环
2. **工具系统** — `Tool.define()` 统一接口
3. **Provider** — 20+ 供应商，Vercel AI SDK 抽象
4. **会话** — SQLite 持久化，MessageV2 消息格式
5. **权限** — 通配符规则，分层合并，Deferred 等待
6. **Prompt** — 每个模型定制，Plugin 可扩展
7. **Skill** — Markdown 指令，多路径扫描，按需加载
8. **MCP** — 标准协议，三种传输，工具转换
9. **LSP** — 编辑器智能，诊断反馈
10. **Plugin** — Hook-based，npm 分发
11. **Snapshot** — Git 快照，Worktree 隔离
12. **C/S 架构** — Hono 服务，Bus 事件，多客户端

**核心洞察：** OpenCode 不只是一个 CLI 工具，它是一个**平台**。Effect-TS 提供了结构化的依赖注入和并发控制，Vercel AI SDK 统一了模型调用，Hono 让它能服务任何客户端。这种架构选择让 OpenCode 在开源世界中独树一帜。
