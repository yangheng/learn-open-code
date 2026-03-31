# S08 — MCP 集成：协议即互操作

> **格言：** "协议即互操作"

[← S07 Skill 系统](s07-skill-system.md) | [S09 LSP 集成 →](s09-lsp-integration.md)

---

## 问题

AI Agent 需要与外部服务交互（数据库、API、专有工具）。如何用标准化的方式集成这些外部能力，而不需要为每个服务写专门的适配器？

## 架构概览

```
┌──────────────────────────────────────────────────┐
│                 MCP 集成                          │
│                                                  │
│  配置 (opencode.json):                            │
│  {                                               │
│    "mcp": {                                      │
│      "my-server": {                              │
│        "type": "stdio" | "sse" | "streamable",   │
│        "command": "node server.js",              │
│        "args": [...],                            │
│        "env": {...}                              │
│      }                                           │
│    }                                             │
│  }                                               │
│                                                  │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐ │
│  │ MCP      │     │ MCP      │     │ MCP      │ │
│  │ Client 1 │     │ Client 2 │     │ Client N │ │
│  │ (stdio)  │     │ (sse)    │     │(http)    │ │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘ │
│       │                │                │        │
│       ▼                ▼                ▼        │
│  ┌───────────────────────────────────────────┐   │
│  │    转换为 AI SDK Tool → 注入 LLM 工具列表  │   │
│  └───────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

## 核心代码

### MCP 状态类型

```typescript
// packages/opencode/src/mcp/index.ts

export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth") }),
  z.object({ status: z.literal("needs_client_registration"), error: z.string() }),
])
```

### 三种传输方式

OpenCode 支持 MCP 的三种标准传输：

```typescript
// packages/opencode/src/mcp/index.ts

import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js"
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js"
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js"
```

- **stdio** — 启动子进程，通过 stdin/stdout 通信
- **SSE** — HTTP Server-Sent Events
- **Streamable HTTP** — 新的双向 HTTP 流

### MCP 工具转换

MCP 工具定义转换为 AI SDK 的 `Tool` 类型：

```typescript
// packages/opencode/src/mcp/index.ts

function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Tool {
  const inputSchema = mcpTool.inputSchema
  const schema: JSONSchema7 = {
    ...(inputSchema as JSONSchema7),
    type: "object",
    properties: (inputSchema.properties ?? {}) as JSONSchema7["properties"],
    additionalProperties: false,
  }

  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      return client.callTool(
        { name: mcpTool.name, arguments: args },
        CallToolResultSchema,
        { timeout: timeout ?? DEFAULT_TIMEOUT }
      )
    },
  })
}
```

### 工具变更通知

MCP 服务器可以动态添加/删除工具：

```typescript
// packages/opencode/src/mcp/index.ts

export const ToolsChanged = BusEvent.define("mcp.tools.changed", z.object({
  server: z.string(),
}))

// 监听 MCP 的 ToolListChangedNotification
// 通过 Bus 事件通知其他组件刷新工具列表
```

### OAuth 认证

MCP 支持 OAuth 认证流程：

```typescript
// packages/opencode/src/mcp/index.ts

type TransportWithAuth = StreamableHTTPClientTransport | SSEClientTransport
const pendingOAuthTransports = new Map<string, TransportWithAuth>()

// 当 MCP 服务器返回 UnauthorizedError 时
// 启动 OAuth 流程 → 打开浏览器 → 等待回调
// 认证完成后重新连接
```

### 资源类型

```typescript
export const Resource = z.object({
  name: z.string(),
  uri: z.string(),
  description: z.string().optional(),
  mimeType: z.string().optional(),
  client: z.string(),
})
```

### MCP 命令集成

MCP 服务器提供的 prompt 可以作为命令使用：

```typescript
// packages/opencode/src/command/index.ts (Command 系统)

// MCP prompts 被注册为 slash commands
// 用户可以通过 /command 方式调用 MCP prompt
```

## 超时控制

```typescript
const DEFAULT_TIMEOUT = 30_000  // 30 秒默认超时

// 每个工具调用都有超时保护
client.callTool(
  { name: mcpTool.name, arguments: args },
  CallToolResultSchema,
  { timeout: timeout ?? DEFAULT_TIMEOUT }
)
```

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| MCP 支持 | ✅ 完整 (stdio/sse/http) | ✅ 完整 |
| OAuth | ✅ 内置 OAuth 流程 | ✅ 支持 |
| 配置方式 | `opencode.json` mcp 字段 | `claude_desktop_config.json` |
| 工具转换 | MCP → AI SDK Tool | MCP → 内部格式 |
| 动态刷新 | ToolListChanged 通知 | 支持 |
| 命令集成 | MCP prompts → slash commands | 类似 |

---

[← S07 Skill 系统](s07-skill-system.md) | [S09 LSP 集成 →](s09-lsp-integration.md)
