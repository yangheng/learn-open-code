# S09 — LSP 集成：编辑器智能，终端享用

> **格言：** "编辑器智能，终端享用"

[← S08 MCP 集成](s08-mcp-integration.md) | [S10 插件系统 →](s10-plugin-system.md)

---

## 问题

AI Agent 修改代码后，如何知道是否引入了类型错误？如何获得跳转定义、查找引用等 IDE 级别的代码理解能力？

## 架构概览

```
┌──────────────────────────────────────────────┐
│              LSP 集成                         │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │typescript│  │  pyright │  │   gopls  │   │
│  │ -language│  │  / ty    │  │          │   │
│  │ -server  │  │          │  │          │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │         │
│       ▼              ▼              ▼         │
│  ┌───────────────────────────────────────┐    │
│  │          LSPClient (per server)       │    │
│  │  • diagnostics  • hover              │    │
│  │  • definition   • references         │    │
│  │  • documentSymbol • workspaceSymbol  │    │
│  └───────────────────────────────────────┘    │
│                    │                          │
│                    ▼                          │
│  ┌───────────────────────────────────────┐    │
│  │          LSP Tool (可选)              │    │
│  │  Agent 直接调用 LSP 操作              │    │
│  └───────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

## 核心代码

### LSP 接口

```typescript
// packages/opencode/src/lsp/index.ts

export interface Interface {
  readonly init: () => Effect.Effect<void>
  readonly status: () => Effect.Effect<Status[]>
  readonly hasClients: (file: string) => Effect.Effect<boolean>
  readonly touchFile: (input: string, waitForDiagnostics?: boolean) => Effect.Effect<void>
  readonly diagnostics: () => Effect.Effect<Record<string, LSPClient.Diagnostic[]>>
  readonly hover: (input: LocInput) => Effect.Effect<any>
  readonly definition: (input: LocInput) => Effect.Effect<any[]>
  readonly references: (input: LocInput) => Effect.Effect<any[]>
}

type LocInput = { file: string; line: number; character: number }
```

### LSP 状态

```typescript
// packages/opencode/src/lsp/index.ts

export const Status = z.object({
  id: z.string(),
  name: z.string(),
  root: z.string(),
  status: z.union([z.literal("connected"), z.literal("error")]),
})
```

### 类型定义

```typescript
// packages/opencode/src/lsp/index.ts

export const Symbol = z.object({
  name: z.string(),
  kind: z.number(),       // Class, Function, Method, etc.
  location: z.object({
    uri: z.string(),
    range: Range,
  }),
})

export const DocumentSymbol = z.object({
  name: z.string(),
  detail: z.string().optional(),
  kind: z.number(),
  range: Range,
  selectionRange: Range,
})

// 过滤的符号类型
const kinds = [
  SymbolKind.Class,
  SymbolKind.Function,
  SymbolKind.Method,
  SymbolKind.Interface,
  SymbolKind.Variable,
  SymbolKind.Constant,
  SymbolKind.Struct,
  SymbolKind.Enum,
]
```

### 实验性 LSP 服务器支持

```typescript
// packages/opencode/src/lsp/index.ts

const filterExperimentalServers = (servers: Record<string, LSPServer.Info>) => {
  // ty (Rust-based Python type checker) 可以替代 pyright
  if (Flag.OPENCODE_EXPERIMENTAL_LSP_TY) {
    if (servers["pyright"]) delete servers["pyright"]
  } else {
    if (servers["ty"]) delete servers["ty"]
  }
}
```

### LSP Tool

通过 feature flag 启用 LSP 工具：

```typescript
// packages/opencode/src/tool/lsp.ts

// Agent 可以直接调用 LSP 操作：
// - 获取某文件的诊断信息（类型错误等）
// - 查找定义
// - 查找引用
// - Hover 信息
```

### 文件修改通知

当 Agent 通过 edit/write 工具修改文件后：

```typescript
// touchFile → 通知 LSP 服务器文件已更改
// → LSP 重新分析
// → diagnostics 更新
// → Agent 可以检查是否引入错误
```

## LSP Server 配置

```typescript
// packages/opencode/src/lsp/server.ts

// 内置的 LSP 服务器配置：
// - TypeScript: typescript-language-server
// - Python: pyright (或实验性 ty)
// - Go: gopls
// 用户可以通过配置添加更多
```

## 关键设计

### 按需启动

LSP 服务器不是一开始就全部启动，而是当 Agent 操作特定文件类型时才启动对应的 LSP 服务器。

### 诊断反馈循环

```
Agent 修改文件 → touchFile() → LSP 分析 → diagnostics
→ Agent 检查是否有错误 → 如有，继续修复
```

### 实验性质

LSP 集成在 OpenCode 中仍是实验性功能，通过 `OPENCODE_EXPERIMENTAL_LSP_TOOL` flag 控制。

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| LSP 集成 | ✅ 内置 (实验性) | ❌ 无 |
| 诊断反馈 | touchFile → diagnostics | 依赖 bash 运行编译器 |
| 代码导航 | definition/references | 依赖 grep/搜索 |
| 语言支持 | TS/Python/Go (可配置) | 无 LSP |

---

[← S08 MCP 集成](s08-mcp-integration.md) | [S10 插件系统 →](s10-plugin-system.md)
