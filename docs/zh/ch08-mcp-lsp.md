# 第八章：外部集成 —— MCP 与 LSP

> **格言**：一个人走得快，一群人走得远。

## 上回说到

系统提示组装完毕，工具集合中包含内置工具和 MCP 工具。本章深入 MCP 和 LSP 这两个外部集成。

## MCP：外部工具服务器

### 1. 什么是 MCP

MCP（Model Context Protocol）让 OpenCode 连接外部工具服务器。配置在 `opencode.json`：

```json
{
  "mcp": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@my-org/mcp-server"]
    }
  }
}
```

### 2. 客户端初始化

```typescript
// src/mcp/index.ts（简化）
// 支持三种传输方式：
// 1. stdio  - 通过子进程的 stdin/stdout
// 2. sse    - Server-Sent Events
// 3. http   - Streamable HTTP
const transport = new StdioClientTransport({ command, args, env })
const client = new Client({ name: "opencode", version: Installation.VERSION })
await client.connect(transport)
```

### 3. 工具发现

MCP 服务器连接后，OpenCode 获取它提供的工具列表：

```typescript
// MCP tools() 返回所有已连接服务器的工具
// 这些工具在 resolveTools() 中与内置工具合并
for (const [key, item] of Object.entries(await MCP.tools())) {
  tools[key] = item  // key 是全局唯一的工具名
}
```

### 4. MCP 工具的权限处理

MCP 工具默认需要用户审批：

```typescript
// src/session/prompt.ts:L432（resolveTools 中 MCP 部分）
item.execute = async (args, opts) => {
  await ctx.ask({
    permission: key,           // 工具名作为权限名
    patterns: ["*"],
    always: ["*"],
  })
  const result = await execute(args, opts)
  return { output: truncated.content, metadata }
}
```

### 5. 资源读取

MCP 还支持资源（Resources），可以在 TUI 中附加到消息：

```typescript
// src/mcp/index.ts
export async function readResource(clientName: string, uri: string) {
  const client = getClient(clientName)
  return client.readResource({ uri })
}
```

### 6. OAuth 认证

远程 MCP 服务器可能需要 OAuth：

```typescript
// src/mcp/oauth-provider.ts
// OpenCode 实现了 MCP SDK 的 OAuthClientProvider 接口
// 支持完整的 OAuth 2.0 授权码流程
// 凭证存储在 ~/.opencode/mcp-auth/ 目录
```

## LSP：代码智能

### 1. 自动检测语言服务器

```typescript
// src/lsp/index.ts（getClients 函数）
for (const server of Object.values(s.servers)) {
  if (server.extensions.length && !server.extensions.includes(extension)) continue
  const root = await server.root(file)
  if (!root) continue
  // 按需启动 LSP 服务器
  const client = await schedule(server, root, key)
  if (client) result.push(client)
}
```

OpenCode 内置了多个 LSP 服务器配置（`src/lsp/server.ts`），包括 TypeScript、Python、Go、Rust 等。

### 2. LSP 在工具系统中的角色

LSP 不直接作为 LLM 工具暴露，而是通过两种方式参与：

**a) `lsp` 工具**：提供符号搜索、跳转定义等

```typescript
// src/tool/lsp.ts（简化）
export const LspTool = Tool.define("lsp", {
  parameters: z.object({
    action: z.enum(["definition", "references", "hover", "diagnostics", ...]),
    file: z.string(),
    line: z.number(),
    character: z.number(),
  }),
  async execute(args, ctx) {
    switch (args.action) {
      case "definition": return LSP.definition({ file, line, character })
      case "references": return LSP.references({ file, line, character })
      // ...
    }
  }
})
```

**b) 文件附加时的符号解析**：

```typescript
// src/session/prompt.ts（createUserMessage 内部）
// 当用户附加文件并指定行号时，LSP 帮助定位完整的符号范围
const symbols = await LSP.documentSymbol(filePathURI)
for (const symbol of symbols) {
  if (symbol.range?.start?.line === start) {
    start = symbol.range.start.line
    end = symbol.range.end.line
    break
  }
}
```

**c) 编辑后的诊断**：

工具修改文件后，LSP 提供诊断信息（错误、警告），帮助 LLM 发现引入的问题。

### 3. 诊断集成

```typescript
// src/lsp/index.ts
export const diagnostics = async () => {
  const results: Record<string, LSPClient.Diagnostic[]> = {}
  const all = yield* runAll(async (client) => client.diagnostics)
  for (const result of all) {
    for (const [p, diags] of result.entries()) {
      results[p] = [...(results[p] || []), ...diags]
    }
  }
  return results
}
```

## 架构图

![ch08-mcp-lsp](../images/ch08-mcp-lsp.webp)

## 关键洞察

1. **MCP 是工具扩展**：外部服务器的工具与内置工具在同一个池子里，LLM 不区分来源
2. **LSP 是被动集成**：不直接参与 LLM 循环，而是通过 lsp 工具和文件诊断间接影响
3. **按需启动**：LSP 服务器在首次访问相关文件时才启动，不浪费资源
4. **MCP 默认需审批**：安全考虑，所有 MCP 工具调用都要用户确认（除非配置 allow）

## 下一章预告

LLM 通过工具修改了代码，但改错了怎么办？Snapshot 系统提供了完美的撤销机制。

---

← [上一章：第七章：提示词组装](./ch07-prompts-skills.md) | [下一章：第九章：快照与撤销](./ch09-snapshots.md) →
