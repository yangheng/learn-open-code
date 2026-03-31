# S04 — 会话管理：对话即数据库行

> **格言：** "对话即数据库行"

[← S03 Provider 系统](s03-provider-system.md) | [S05 权限系统 →](s05-permission-system.md)

---

## 问题

AI Agent 需要跟踪对话历史、消息、工具调用结果、元数据。这些数据如何组织、存储、查询？

## 架构概览

```
┌───────────────────────────────────────────┐
│              Session 系统                  │
│                                           │
│  Session (会话)                            │
│  ├── id, slug, projectID                  │
│  ├── title, version                       │
│  ├── parentID (子会话链)                   │
│  ├── summary (additions/deletions/files)  │
│  └── time (created/updated/compacting)    │
│                                           │
│  MessageV2 (消息)                          │
│  ├── User     { text, system, variant }   │
│  ├── Assistant { model, agent, cost }     │
│  └── Parts:                               │
│      ├── TextPart      → 文本输出         │
│      ├── ReasoningPart → 思考过程         │
│      ├── ToolPart      → 工具调用+结果    │
│      └── FilePart      → 文件附件         │
│                                           │
│  存储: SQLite (Drizzle ORM)               │
└───────────────────────────────────────────┘
```

## 核心代码

### Session.Info — 会话数据结构

```typescript
// packages/opencode/src/session/index.ts

export const Info = z.object({
  id: SessionID.zod,
  slug: z.string(),
  projectID: ProjectID.zod,
  workspaceID: WorkspaceID.zod.optional(),
  directory: z.string(),
  parentID: SessionID.zod.optional(),      // 父子会话关系
  summary: z.object({
    additions: z.number(),
    deletions: z.number(),
    files: z.number(),
    diffs: Snapshot.FileDiff.array().optional(),
  }).optional(),
  share: z.object({ url: z.string() }).optional(),
  title: z.string(),
  version: z.string(),
  revert: /* ... */,
  permission: /* ... */,
  time: z.object({
    created: z.number(),
    updated: z.number(),
    compacting: z.number().optional(),
    archived: z.number().optional(),
  }),
})
```

### MessageV2 — 消息系统

OpenCode 使用 `MessageV2` 作为消息格式，每条消息由多个 Part 组成：

```typescript
// packages/opencode/src/session/message-v2.ts

// 文本部分
export type TextPart = {
  id: PartID; type: "text"
  text: string
}

// 推理/思考部分
export type ReasoningPart = {
  id: PartID; type: "reasoning"
  text: string
  time: { start: number; end?: number }
  metadata?: any
}

// 工具调用部分
export type ToolPart = {
  id: PartID; type: "tool"
  tool: string
  callID: string
  state:
    | { status: "pending"; input: any; raw: string }
    | { status: "running"; input: any; time: { start: number } }
    | { status: "completed"; input: any; output: string; time: { start: number; end: number } }
    | { status: "error"; input: any; error: string; time: { start: number; end: number } }
  metadata?: any
}

// 文件附件部分
export type FilePart = {
  id: PartID; type: "file"
  mediaType: string
  filename?: string
  url: string
}
```

### 父子会话 — 子 Agent 的实现基础

当 `task` 工具创建子 Agent 时，实际上是创建一个新 Session，并设置 `parentID`：

```typescript
// packages/opencode/src/tool/task.ts

const session = await Session.create({
  parentID: ctx.sessionID,
  title: params.description + ` (@${agent.name} subagent)`,
})
```

### Session 上下文压缩 (Compaction)

当对话过长时，OpenCode 会自动压缩上下文：

```typescript
// packages/opencode/src/session/compaction.ts

// 使用 "compaction" agent 来总结对话历史
// processor 返回 "compact" 时触发
// 压缩后的摘要替代原始消息，减少 token 消耗
```

### 会话恢复与 Fork

```typescript
// packages/opencode/src/session/index.ts

function getForkedTitle(title: string): string {
  const match = title.match(/^(.+) \(fork #(\d+)\)$/)
  if (match) {
    const base = match[1]
    const num = parseInt(match[2], 10)
    return `${base} (fork #${num + 1})`
  }
  return `${title} (fork #1)`
}
```

## 数据库 Schema

会话和消息存储在 SQLite 中，使用 Drizzle ORM：

```typescript
// packages/opencode/src/session/session.sql.ts

// SessionTable 包含:
// - id, slug, project_id, workspace_id
// - parent_id, directory, title, version
// - summary_additions, summary_deletions, summary_files, summary_diffs
// - share_url, revert, permission
// - time_created, time_updated, time_compacting, time_archived
```

## 消息转换

OpenCode 在内部消息格式和 AI SDK 消息格式之间进行转换，确保工具调用历史被正确传递给 LLM。

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 存储 | SQLite (Drizzle ORM) | 本地 JSON 文件 |
| 消息格式 | MessageV2 多部分结构 | 自有格式 |
| 子会话 | parentID 链接 | 无 |
| 上下文压缩 | 自动 compaction | 手动管理 |
| 会话 Fork | 内置支持 | 无 |
| 结构化查询 | SQL 查询 | 文件系统 |

---

[← S03 Provider 系统](s03-provider-system.md) | [S05 权限系统 →](s05-permission-system.md)
