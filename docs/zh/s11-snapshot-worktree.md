# S11 — 快照与工作树：大胆修改，随时回滚

> **格言：** "大胆修改，随时回滚"

[← S10 插件系统](s10-plugin-system.md) | [S12 客户端/服务端 →](s12-client-server.md)

---

## 问题

AI Agent 修改代码时可能犯错。如何让用户安全地回滚任意更改？如何让多个 Agent 并行修改代码而不互相干扰？

## 架构概览

```
┌──────────────────────────────────────────────────┐
│           Snapshot 系统 (Git-based)               │
│                                                  │
│  每次工具调用前:                                   │
│    snapshot.track() → git add -A → git commit     │
│                                                  │
│  回滚:                                            │
│    snapshot.restore(hash) → git checkout           │
│    snapshot.revert(patches) → git apply --reverse  │
│                                                  │
│  Diff:                                            │
│    snapshot.diff(hash) → git diff                  │
│    snapshot.diffFull(from, to) → FileDiff[]        │
│                                                  │
│  存储: $DATA/snapshot/$projectID/$hash/            │
│  独立 git 仓库，不影响用户的 git 历史              │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│           Worktree 系统 (Git Worktree)            │
│                                                  │
│  创建隔离的工作目录:                               │
│    git worktree add <path> -b <branch>            │
│                                                  │
│  每个子 Agent 可以在独立 worktree 中工作           │
│  修改完成后合并回主分支                            │
│                                                  │
│  生命周期:                                        │
│    create → (agent works) → reset/remove          │
└──────────────────────────────────────────────────┘
```

## 核心代码

### Snapshot 接口

```typescript
// packages/opencode/src/snapshot/index.ts

export interface Interface {
  readonly init: () => Effect.Effect<void>
  readonly cleanup: () => Effect.Effect<void>
  readonly track: () => Effect.Effect<string | undefined>     // 创建快照
  readonly patch: (hash: string) => Effect.Effect<Snapshot.Patch>
  readonly restore: (snapshot: string) => Effect.Effect<void> // 回滚到某快照
  readonly revert: (patches: Snapshot.Patch[]) => Effect.Effect<void>
  readonly diff: (hash: string) => Effect.Effect<string>
  readonly diffFull: (from: string, to: string) => Effect.Effect<Snapshot.FileDiff[]>
}
```

### 核心数据类型

```typescript
// packages/opencode/src/snapshot/index.ts

export const Patch = z.object({
  hash: z.string(),
  files: z.string().array(),
})

export const FileDiff = z.object({
  file: z.string(),
  before: z.string(),
  after: z.string(),
  additions: z.number(),
  deletions: z.number(),
  status: z.enum(["added", "deleted", "modified"]).optional(),
})
```

### 独立 Git 仓库

Snapshot 使用独立的 Git 仓库，与用户项目的 Git 完全分离：

```typescript
// packages/opencode/src/snapshot/index.ts

const state = {
  directory: ctx.directory,
  worktree: ctx.worktree,
  // 独立的 git 目录，不是项目的 .git
  gitdir: path.join(Global.Path.data, "snapshot", ctx.project.id, Hash.fast(ctx.worktree)),
  vcs: ctx.project.vcs,
}

// 所有 git 命令使用这个独立目录
const args = (cmd: string[]) => [
  "--git-dir", state.gitdir,
  "--work-tree", state.worktree,
  ...cmd
]
```

### Git 配置

```typescript
// packages/opencode/src/snapshot/index.ts

const core = ["-c", "core.longpaths=true", "-c", "core.symlinks=true"]
const cfg = ["-c", "core.autocrlf=false", ...core]
const quote = [...cfg, "-c", "core.quotepath=false"]
```

### 排除文件同步

Snapshot 复用项目的 `.gitignore` 排除规则：

```typescript
const sync = Effect.fnUntraced(function* (list: string[] = []) {
  const file = yield* excludes()  // 项目的 .gitignore
  const target = path.join(state.gitdir, "info", "exclude")
  const text = [
    file ? (yield* read(file)).trimEnd() : "",
    ...list.map(item => `/${item.replaceAll("\\", "/")}`),
  ].filter(Boolean).join("\n")
  // 写入快照仓库的排除文件
  yield* fs.writeFileString(target, text ? `${text}\n` : "")
})
```

### 文件大小限制

```typescript
const limit = 2 * 1024 * 1024  // 2MB — 超过此大小的文件不追踪
```

### 自动清理

```typescript
const prune = "7.days"  // 7 天前的快照自动清理
```

### Worktree 数据类型

```typescript
// packages/opencode/src/worktree/index.ts

export const Info = z.object({
  name: z.string(),
  branch: z.string(),
  directory: z.string(),
})

export const CreateInput = z.object({
  name: z.string().optional(),
  startCommand: z.string().optional()
    .describe("Additional startup script to run after the project's start command"),
})
```

### Worktree 事件

```typescript
// packages/opencode/src/worktree/index.ts

export const Event = {
  Ready: BusEvent.define("worktree.ready", z.object({
    name: z.string(),
    branch: z.string(),
  })),
  Failed: BusEvent.define("worktree.failed", z.object({
    message: z.string(),
  })),
}
```

## 在 Agent 循环中的使用

```typescript
// packages/opencode/src/session/processor.ts

// 每次工具调用前创建快照
// ctx.snapshot = await snapshot.track()

// 每次会话结束后生成 diff 摘要
// session.summary = { additions, deletions, files }
```

## 并发锁

```typescript
// 使用 Semaphore 防止并发 git 操作冲突
const lock = (key: string) => {
  const hit = locks.get(key)
  if (hit) return hit
  const next = Semaphore.makeUnsafe(1)
  locks.set(key, next)
  return next
}

const locked = <A, E, R>(fx: Effect.Effect<A, E, R>) =>
  lock(state.gitdir).withPermits(1)(fx)
```

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 回滚机制 | 独立 Git 快照仓库 | Checkpoint (文件副本) |
| 粒度 | 每次工具调用前 | 每次对话轮次 |
| Diff 查看 | `diffFull()` → FileDiff[] | 简单 diff |
| 并行隔离 | Git worktree | 无 |
| 自动清理 | 7 天过期 | 手动 |
| 文件限制 | 2MB 上限 | 未知 |

---

[← S10 插件系统](s10-plugin-system.md) | [S12 客户端/服务端 →](s12-client-server.md)
