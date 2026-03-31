# 第九章：快照与撤销 —— Snapshot 系统

> **格言**：勇敢修改的前提，是随时能撤回。

## 上回说到

MCP 和 LSP 扩展了 OpenCode 的能力。但无论多强大的工具，都可能犯错。Snapshot 系统是安全网。

## 代码路径

### 1. 每个 Step 开始时创建快照

回到 Processor 的事件处理：

```typescript
// src/session/processor.ts:L136（handleEvent 中 start-step 分支）
case "start-step":
  ctx.snapshot = yield* snapshot.track()
  yield* session.updatePart({
    type: "step-start",
    snapshot: ctx.snapshot,
  })
```

`snapshot.track()` 在**工具执行前**记录文件系统状态。

### 2. Snapshot 实现：Shadow Git Repo

Snapshot 不是简单的文件拷贝，而是使用一个**影子 Git 仓库**：

```typescript
// src/snapshot/index.ts（简化）
// 影子仓库位于 ~/.opencode/data/snapshot/<project-hash>/
// 每次 track() 做的事：
//   git add -A
//   git commit -m "snapshot"
//   返回 commit hash

const track = Effect.fn("Snapshot.track")(function* () {
  yield* git([...cfg, "add", "-A", "."], { cwd: worktree })
  const result = yield* git(
    [...cfg, "commit", "--allow-empty", "-m", "snapshot", "--no-verify"],
    { cwd: worktree },
  )
  const hash = yield* git(["rev-parse", "HEAD"], { cwd: worktree })
  return hash.text.trim()
})
```

### 3. Step 结束时计算 Patch

```typescript
// src/session/processor.ts:L160（handleEvent 中 finish-step 分支）
case "finish-step":
  if (ctx.snapshot) {
    const patch = yield* snapshot.patch(ctx.snapshot)
    if (patch.files.length) {
      yield* session.updatePart({
        type: "patch",
        hash: patch.hash,
        files: patch.files,  // 本次 step 修改的文件列表
      })
    }
  }
```

`patch()` 对比快照和当前状态：

```typescript
// src/snapshot/index.ts（简化）
const patch = Effect.fn("Snapshot.patch")(function* (hash: string) {
  yield* git([...cfg, "add", "-A", "."], { cwd: worktree })
  yield* git([...cfg, "commit", "--allow-empty", "-m", "snapshot"], { cwd: worktree })
  const current = yield* git(["rev-parse", "HEAD"], { cwd: worktree })
  const diff = yield* git([...quote, "diff", "--name-only", hash, current.text.trim()], { cwd: worktree })
  return {
    hash: current.text.trim(),
    files: diff.text.split("\n").filter(Boolean),
  }
})
```

### 4. 撤销操作

用户在 TUI 中可以撤销任何一个 step：

```typescript
// src/snapshot/index.ts
const restore = Effect.fn("Snapshot.restore")(function* (snapshotHash: string) {
  // 将工作区恢复到指定快照的状态
  yield* git([...cfg, "checkout", snapshotHash, "--", "."], { cwd: worktree })
})

const revert = Effect.fn("Snapshot.revert")(function* (patches: Patch[]) {
  // 逆序还原多个 patch
  for (const patch of patches.reverse()) {
    yield* restore(patch.hash)
  }
})
```

### 5. Session 级别的 Diff

Snapshot 还提供整个 Session 的 diff 视图：

```typescript
// src/snapshot/index.ts
const diffFull = Effect.fn("Snapshot.diffFull")(function* (from: string, to: string) {
  const result = yield* git([...quote, "diff", from, to], { cwd: worktree })
  // 解析 diff 输出为结构化的 FileDiff[]
  return parseDiff(result.text)
})
```

这让 TUI 能显示 "本次对话修改了哪些文件，增删了多少行"。

### 6. 自动清理

影子仓库会自动清理过期的快照：

```typescript
// 配置: prune = "7.days"
// 超过 7 天的快照自动清理
// 单个文件超过 2MB 不跟踪
```

## 架构图

![ch09-snapshots](../images/ch09-snapshots.webp)

## 关键洞察

1. **Git 作为快照引擎**：不重新发明轮子，用 Git 的增量存储实现高效快照
2. **Step 粒度**：每个 LLM 回复的每个 step（可能包含多个工具调用）一个快照
3. **Patch 记录变更**：不仅能撤销，还能查看每个 step 改了什么
4. **影子仓库隔离**：不污染项目的 Git 历史

## 下一章预告

到目前为止，我们跟踪的都是单个 Session 在单个工作目录中的流程。但 OpenCode 支持 Worktree——在独立的 Git 工作树中并行执行多个任务。最后，为什么 OpenCode 是一个 Server 而不只是 CLI？

---

← [上一章：第八章：外部集成](./ch08-mcp-lsp.md) | [下一章：第十章：并行与架构](./ch10-architecture.md) →
