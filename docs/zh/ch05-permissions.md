# 第五章：权限 —— 用户批准工具调用

> **格言**：信任是分层的，权限也是。

## 上回说到

`bash` 工具调用了 `ctx.ask({ permission: "bash", patterns: [args.command] })`。这条请求进入了权限系统。

## 代码路径

### 1. Permission.ask()：入口

```typescript
// src/permission/index.ts:L126（ask 函数）
const ask = Effect.fn("Permission.ask")(function* (input) {
  const { approved, pending } = yield* InstanceState.get(state)
  const { ruleset, ...request } = input
  let needsAsk = false

  for (const pattern of request.patterns) {
    const rule = evaluate(request.permission, pattern, ruleset, approved)
    if (rule.action === "deny") {
      return yield* new DeniedError({ ruleset: ... })
    }
    if (rule.action === "allow") continue
    needsAsk = true
  }

  if (!needsAsk) return  // 所有 pattern 都 allow，直接通过

  // 需要用户确认
  const deferred = yield* Deferred.make<void, RejectedError | CorrectedError>()
  pending.set(id, { info, deferred })
  void Bus.publish(Event.Asked, info)     // 发布事件，通知 TUI
  return yield* Deferred.await(deferred)  // 阻塞，等待用户回复
})
```

关键设计：**`Deferred.await()`** 让工具执行暂停，直到用户通过 TUI 回复。

### 2. 规则求值：三层匹配

```typescript
// src/permission/evaluate.ts
export function evaluate(permission: string, pattern: string, ...rulesets: Rule[][]): Rule {
  const rules = rulesets.flat()
  const match = rules.findLast(
    (rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern),
  )
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

规则匹配使用 **findLast**——后面的规则覆盖前面的。规则来源（从低到高优先级）：

1. **Agent 默认规则**：`build` Agent 的 `{ "*": "allow", doom_loop: "ask" }`
2. **用户配置规则**：`opencode.json` 中的 `permission` 字段
3. **Session 运行时规则**：`Permission.reply("always")` 产生的临时规则

### 3. 用户回复

TUI 收到 `permission.asked` 事件后展示审批对话框。用户有三个选择：

```typescript
// src/permission/index.ts:L155（reply 函数）
const reply = Effect.fn("Permission.reply")(function* (input) {
  const existing = pending.get(input.requestID)

  if (input.reply === "reject") {
    yield* Deferred.fail(existing.deferred, new RejectedError())
    // 同时拒绝同 session 的所有 pending 请求
    for (const [id, item] of pending.entries()) {
      if (item.info.sessionID !== existing.info.sessionID) continue
      yield* Deferred.fail(item.deferred, new RejectedError())
    }
    return
  }

  yield* Deferred.succeed(existing.deferred, undefined)  // 放行

  if (input.reply === "once") return  // 只这一次

  // "always" → 将 pattern 加入 approved 列表
  for (const pattern of existing.info.always) {
    approved.push({ permission: existing.info.permission, pattern, action: "allow" })
  }

  // 自动放行同 session 中匹配新规则的 pending 请求
  for (const [id, item] of pending.entries()) {
    const ok = item.info.patterns.every(
      (pattern) => evaluate(item.info.permission, pattern, approved).action === "allow",
    )
    if (ok) yield* Deferred.succeed(item.deferred, undefined)
  }
})
```

三种回复：
- **once**：只放行这一次
- **always**：将此 pattern 加入白名单（本次项目会话期间有效）
- **reject**：拒绝，并中止同 session 的所有 pending 请求

### 4. Doom Loop 检测

Processor 还有一个特殊的权限检查——防止 LLM 陷入死循环：

```typescript
// src/session/processor.ts:L119（handleEvent 中 tool-call 分支）
const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)  // DOOM_LOOP_THRESHOLD = 3

if (
  recentParts.length === DOOM_LOOP_THRESHOLD &&
  recentParts.every(
    (part) =>
      part.type === "tool" &&
      part.tool === value.toolName &&
      JSON.stringify(part.state.input) === JSON.stringify(value.input),
  )
) {
  // 连续 3 次相同的工具调用 → 请求用户干预
  yield* permission.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    metadata: { tool: value.toolName, input: value.input },
  })
}
```

### 5. run 模式的自动拒绝

在 `opencode run` 非交互模式下，没有用户可以审批。所以所有权限请求都被自动拒绝：

```typescript
// src/cli/cmd/run.ts:L445（execute 内部）
if (event.type === "permission.asked") {
  await sdk.permission.reply({ requestID: permission.id, reply: "reject" })
}
```

并且 `run` 命令预设了严格的权限规则：

```typescript
// src/cli/cmd/run.ts:L340
const rules: Permission.Ruleset = [
  { permission: "question", action: "deny", pattern: "*" },
  { permission: "plan_enter", action: "deny", pattern: "*" },
  { permission: "plan_exit", action: "deny", pattern: "*" },
]
```

## 架构图

![ch05-permissions](../images/ch05-permissions.webp)

## 关键洞察

1. **Deferred 实现异步审批**：工具执行暂停在 `Deferred.await()`，用户回复后才继续
2. **规则是 findLast 语义**：后声明的规则覆盖先声明的，允许精确覆盖
3. **always 有连锁效应**：批准一个 pattern 后，同 session 的所有匹配请求自动通过
4. **reject 是广谱的**：拒绝一个请求时，同 session 的所有 pending 请求都被拒绝

## 下一章预告

随着对话的进行，消息历史越来越长。当 token 数接近模型上下文限制时，会发生什么？

---

← [上一章：第四章：工具调度](./ch04-tools.md) | [下一章：第六章：上下文管理](./ch06-compaction.md) →
