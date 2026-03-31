# S10 — 插件系统：钩子驱动，无限扩展

> **格言：** "钩子驱动，无限扩展"

[← S09 LSP 集成](s09-lsp-integration.md) | [S11 快照与工作树 →](s11-snapshot-worktree.md)

---

## 问题

如何在不修改核心代码的情况下扩展 Agent 的能力？如何让第三方开发者为 OpenCode 添加新的认证方式、工具转换、参数调整？

## 架构概览

```
┌───────────────────────────────────────────────┐
│              Plugin 系统                       │
│                                               │
│  内置插件:                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Copilot  │  │  Codex   │  │  GitLab  │    │
│  │  Auth    │  │  Auth    │  │  Auth    │    │
│  └──────────┘  └──────────┘  └──────────┘    │
│  ┌──────────┐                                 │
│  │  Poe     │                                 │
│  │  Auth    │                                 │
│  └──────────┘                                 │
│                                               │
│  用户插件:                                     │
│  npm 包 / 本地路径 → install → load            │
│                                               │
│  Hooks (触发点):                               │
│  • tool.definition          工具定义转换       │
│  • chat.params              LLM 参数调整       │
│  • chat.headers             请求头注入         │
│  • experimental.chat.system.transform          │
│                             系统提示词转换     │
└───────────────────────────────────────────────┘
```

## 核心代码

### Plugin 接口

```typescript
// packages/opencode/src/plugin/index.ts

export interface Interface {
  readonly trigger: <Name extends TriggerName, Input, Output>(
    name: Name, input: Input, output: Output,
  ) => Effect.Effect<Output>
  readonly list: () => Effect.Effect<Hooks[]>
  readonly init: () => Effect.Effect<void>
}

// Hook 触发模式: (input, output) => Promise<void>
// output 是 mutable 的 —— 插件直接修改 output 对象
type TriggerName = {
  [K in keyof Hooks]-?: NonNullable<Hooks[K]> extends
    (input: any, output: any) => Promise<void> ? K : never
}[keyof Hooks]
```

### 内置认证插件

```typescript
// packages/opencode/src/plugin/index.ts

const INTERNAL_PLUGINS: PluginInstance[] = [
  CodexAuthPlugin,       // OpenAI Codex 认证
  CopilotAuthPlugin,     // GitHub Copilot 认证
  GitlabAuthPlugin,      // GitLab 认证
  PoeAuthPlugin,         // Poe 认证
]
```

### Copilot 认证插件示例

```typescript
// packages/opencode/src/plugin/copilot.ts

// 使用 GitHub Copilot 的 OAuth device flow
// 1. 发起设备认证请求
// 2. 用户在浏览器中授权
// 3. 获取 token
// 4. 注入到请求头
```

### 用户插件加载

```typescript
// packages/opencode/src/plugin/index.ts

async function prepPlugin(item: Config.PluginSpec): Promise<Loaded | undefined> {
  const spec = Config.pluginSpecifier(item)
  if (isDeprecatedPlugin(spec)) return

  const resolved = await resolvePlugin(spec)
  if (!resolved) return

  // npm 插件兼容性检查
  const source = pluginSource(spec)
  if (source === "npm") {
    await checkPluginCompatibility(resolved, Installation.VERSION)
  }

  // 加载入口文件
  const entry = await resolvePluginEntrypoint(spec, target, "server")
  const mod = await import(entry)
  return { item, spec, target, source, mod }
}
```

### Hook 触发机制

```typescript
// packages/opencode/src/plugin/index.ts

const trigger = Effect.fn("Plugin.trigger")(function* (name, input, output) {
  const { hooks } = yield* InstanceState.get(cache)
  for (const hook of hooks) {
    const fn = hook[name]
    if (!fn) continue
    yield* Effect.promise(() => fn(input as any, output as any))
  }
  return output
})
```

关键特点：**output 是可变的**。每个插件直接修改 output 对象，链式处理：

```typescript
// 使用示例 (在 LLM 层)

// 工具定义可被插件修改
const output = { description: next.description, parameters: next.parameters }
yield* plugin.trigger("tool.definition", { toolID: tool.id }, output)
// output.description 和 output.parameters 可能已被插件修改

// LLM 参数可被插件调整
const params = await Plugin.trigger("chat.params", { model, agent }, {
  temperature: 0.7,
  topP: undefined,
  options: {},
})

// 系统提示词可被插件转换
await Plugin.trigger("experimental.chat.system.transform", { model }, { system })
```

### 插件配置

```json
// opencode.json
{
  "plugin": [
    "my-opencode-plugin",              // npm 包
    ["my-plugin", { "key": "value" }], // 带选项的 npm 包
    "./local-plugin"                    // 本地路径
  ]
}
```

### 插件兼容性检查

```typescript
// packages/opencode/src/plugin/shared.ts

// 检查插件的 @opencode-ai/plugin 依赖版本
// 与当前 OpenCode 版本是否兼容
// 不兼容则跳过加载
```

## 自定义工具（Plugin 方式）

插件可以注册新工具：

```typescript
// 插件导出 tool 对象
export const tool = {
  myTool: {
    description: "My custom tool",
    args: { input: z.string() },
    async execute(args, ctx) {
      return "result"
    },
  },
}
```

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 插件系统 | ✅ Hook-based | ❌ 无 |
| 认证扩展 | Copilot/Codex/GitLab/Poe | 仅 API key |
| 工具扩展 | Plugin + 自定义 .ts | 无 |
| 参数调整 | chat.params hook | 无 |
| Prompt 修改 | system.transform hook | 无 |
| 分发方式 | npm / 本地路径 | N/A |

---

[← S09 LSP 集成](s09-lsp-integration.md) | [S11 快照与工作树 →](s11-snapshot-worktree.md)
