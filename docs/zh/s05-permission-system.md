# S05 — 权限系统：通配符规则，分层合并

> **格言：** "通配符规则，分层合并"

[← S04 会话管理](s04-session-management.md) | [S06 提示词工程 →](s06-prompt-engineering.md)

---

## 问题

AI Agent 能执行 shell 命令、修改文件、访问网络。如何控制哪些操作自动执行、哪些需要用户确认、哪些被完全禁止？

## 架构概览

```
┌─────────────────────────────────────────────────┐
│              Permission 系统                     │
│                                                 │
│  Rule = { permission, pattern, action }         │
│  action = "allow" | "deny" | "ask"              │
│                                                 │
│  Ruleset = Rule[]  (有序规则列表)                │
│                                                 │
│  合并优先级:                                     │
│  defaults → agent-specific → user config        │
│                                                 │
│  evaluate(permission, pattern, ...rulesets)      │
│  → 最后匹配的规则胜出                            │
│                                                 │
│  运行时:                                         │
│  Permission.ask() → Deferred → UI 展示           │
│  → 用户回复 once/always/reject                   │
│  → Deferred.resolve/fail                        │
└─────────────────────────────────────────────────┘
```

## 核心代码

### 规则定义

```typescript
// packages/opencode/src/permission/index.ts

export const Action = z.enum(["allow", "deny", "ask"])
export type Action = "allow" | "deny" | "ask"

export const Rule = z.object({
  permission: z.string(),   // 权限类别: "bash", "edit", "read", ...
  pattern: z.string(),      // 通配符模式: "*", "*.env", "src/**"
  action: Action,           // 动作
})

export const Ruleset = Rule.array()
```

### 默认权限配置

每个 Agent 的默认权限在 `agent.ts` 中定义：

```typescript
// packages/opencode/src/agent/agent.ts

const defaults = Permission.fromConfig({
  "*": "allow",                    // 默认允许所有
  doom_loop: "ask",                // doom loop 需确认
  external_directory: {
    "*": "ask",                    // 外部目录需确认
    ...Object.fromEntries(
      whitelistedDirs.map(dir => [dir, "allow"])
    ),
  },
  question: "deny",               // 默认禁止提问
  plan_enter: "deny",
  plan_exit: "deny",
  read: {
    "*": "allow",
    "*.env": "ask",               // .env 文件需确认
    "*.env.*": "ask",
    "*.env.example": "allow",     // .env.example 允许
  },
})
```

**explore** Agent 更严格 —— 只允许读取操作：

```typescript
explore: {
  permission: Permission.merge(defaults, Permission.fromConfig({
    "*": "deny",              // 禁止所有
    grep: "allow",            // 只允许搜索
    glob: "allow",
    list: "allow",
    bash: "allow",
    webfetch: "allow",
    websearch: "allow",
    codesearch: "allow",
    read: "allow",
  })),
}
```

### 权限评估

```typescript
// packages/opencode/src/permission/index.ts

export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule {
  // 遍历所有规则集，使用通配符匹配
  // 最后匹配的规则胜出（后面的覆盖前面的）
  return evalRule(permission, pattern, ...rulesets)
}
```

### 运行时权限请求

当工具需要权限时，通过 `Permission.ask()` 发起请求：

```typescript
// packages/opencode/src/permission/index.ts

const ask = Effect.fn("Permission.ask")(function* (input) {
  const { approved, pending } = yield* InstanceState.get(state)
  const { ruleset, ...request } = input

  for (const pattern of request.patterns) {
    const rule = evaluate(request.permission, pattern, ruleset, approved)
    if (rule.action === "deny") {
      return yield* new DeniedError({ ruleset: /* ... */ })
    }
    if (rule.action === "allow") continue
    needsAsk = true
  }

  if (!needsAsk) return

  // 创建 Deferred，等待用户回复
  const deferred = yield* Deferred.make<void, RejectedError | CorrectedError>()
  pending.set(id, { info, deferred })
  void Bus.publish(Event.Asked, info)  // 通知 UI
  return yield* Deferred.await(deferred)  // 阻塞等待
})
```

### 用户回复处理

```typescript
// packages/opencode/src/permission/index.ts

export const Reply = z.enum(["once", "always", "reject"])

// "once"   → 仅本次允许
// "always" → 记住此规则（持久化到数据库）
// "reject" → 拒绝，工具调用失败
```

### 错误类型

```typescript
export class RejectedError extends Schema.TaggedErrorClass<RejectedError>()(
  "PermissionRejectedError", {}
) {
  get message() { return "The user rejected permission to use this specific tool call." }
}

export class CorrectedError extends Schema.TaggedErrorClass<CorrectedError>()(
  "PermissionCorrectedError", { feedback: Schema.String }
) {
  get message() {
    return `The user rejected permission with feedback: ${this.feedback}`
  }
}

export class DeniedError extends Schema.TaggedErrorClass<DeniedError>()(
  "PermissionDeniedError", { ruleset: Schema.Any }
) {
  get message() {
    return `A rule prevents this tool call. Rules: ${JSON.stringify(this.ruleset)}`
  }
}
```

## 关键设计

### 分层合并

权限规则通过 `Permission.merge()` 分层合并：

```
defaults (基础规则)
   ↓ merge
agent-specific (Agent 特定规则)
   ↓ merge
user config (用户配置)
   ↓ merge
session approved (运行时批准)
```

### 通配符匹配

使用 `Wildcard.match()` 支持 glob 风格的模式匹配：
- `*` 匹配任何单个段
- `**` 匹配任何深度
- `*.env` 匹配所有 .env 文件

### Deferred 模式

使用 Effect-TS 的 `Deferred` 实现异步等待用户输入，这比回调或 Promise 更结构化。

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 规则模型 | 通配符规则 + 分层合并 | 基于操作类型 |
| 配置方式 | 声明式 config | 运行时选择 |
| 粒度 | 文件级别 (pattern matching) | 工具级别 |
| 持久化 | SQLite (always 规则) | 会话级别 |
| Agent 隔离 | 每个 Agent 独立权限集 | 单一权限 |

---

[← S04 会话管理](s04-session-management.md) | [S06 提示词工程 →](s06-prompt-engineering.md)
