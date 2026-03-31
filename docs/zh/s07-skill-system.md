# S07 — Skill 系统：Markdown 即能力

> **格言：** "Markdown 即能力"

[← S06 提示词工程](s06-prompt-engineering.md) | [S08 MCP 集成 →](s08-mcp-integration.md)

---

## 问题

Agent 不可能对所有领域都精通。如何让用户（或社区）为 Agent 添加特定领域的知识和指令，而不需要修改源码？

## 架构概览

```
┌──────────────────────────────────────────────────┐
│                Skill 系统                         │
│                                                  │
│  扫描路径 (优先级从低到高):                        │
│  1. ~/.claude/skills/**/SKILL.md    (兼容)        │
│  2. ~/.agents/skills/**/SKILL.md    (兼容)        │
│  3. .opencode/skills/**/SKILL.md    (项目级)      │
│  4. config.skills.paths             (自定义)      │
│  5. config.skills.urls              (远程拉取)    │
│                                                  │
│  SKILL.md 格式:                                   │
│  ---                                             │
│  name: my-skill                                  │
│  description: Does something useful              │
│  ---                                             │
│  # Instructions                                  │
│  When the user asks to ..., do ...               │
│                                                  │
│  加载方式: skill 工具 → 读取 content → 注入 prompt│
└──────────────────────────────────────────────────┘
```

## 核心代码

### Skill 数据结构

```typescript
// packages/opencode/src/skill/index.ts

export const Info = z.object({
  name: z.string(),
  description: z.string(),
  location: z.string(),    // 文件绝对路径
  content: z.string(),     // Markdown 正文（去掉 frontmatter）
})
```

### 多路径扫描

Skill 从多个位置加载，支持 Claude Code 兼容的目录结构：

```typescript
// packages/opencode/src/skill/index.ts

const EXTERNAL_DIRS = [".claude", ".agents"]  // 兼容 Claude Code / Agents
const EXTERNAL_SKILL_PATTERN = "skills/**/SKILL.md"
const OPENCODE_SKILL_PATTERN = "{skill,skills}/**/SKILL.md"

const loadSkills = Effect.fnUntraced(function* (state, config, discovery, bus, directory, worktree) {
  // 1. 全局目录: ~/.claude/skills/, ~/.agents/skills/
  if (!Flag.OPENCODE_DISABLE_EXTERNAL_SKILLS) {
    for (const dir of EXTERNAL_DIRS) {
      const root = path.join(Global.Path.home, dir)
      yield* scan(state, bus, root, EXTERNAL_SKILL_PATTERN, { dot: true })
    }

    // 2. 项目向上查找
    for await (const root of Filesystem.up({ targets: EXTERNAL_DIRS, start: directory, stop: worktree })) {
      yield* scan(state, bus, root, EXTERNAL_SKILL_PATTERN, { dot: true })
    }
  }

  // 3. .opencode/ 配置目录
  const configDirs = yield* config.directories()
  for (const dir of configDirs) {
    yield* scan(state, bus, dir, OPENCODE_SKILL_PATTERN)
  }

  // 4. 用户配置的路径
  for (const item of cfg.skills?.paths ?? []) {
    const dir = path.isAbsolute(expanded) ? expanded : path.join(directory, expanded)
    yield* scan(state, bus, dir, SKILL_PATTERN)
  }

  // 5. 远程 URL 拉取
  for (const url of cfg.skills?.urls ?? []) {
    const pulledDirs = yield* discovery.pull(url)
    for (const dir of pulledDirs) {
      yield* scan(state, bus, dir, SKILL_PATTERN)
    }
  }
})
```

### Markdown Frontmatter 解析

```typescript
// packages/opencode/src/config/markdown.ts (ConfigMarkdown.parse)

// 解析 SKILL.md 的 YAML frontmatter:
// ---
// name: my-skill
// description: Does X
// ---
// Actual instructions content here...
```

### 权限过滤

Skill 可以通过 Agent 权限过滤：

```typescript
// packages/opencode/src/skill/index.ts

const available = Effect.fn("Skill.available")(function* (agent?: Agent.Info) {
  const s = yield* InstanceState.get(state)
  const list = Object.values(s.skills).toSorted((a, b) => a.name.localeCompare(b.name))
  if (!agent) return list
  return list.filter(skill =>
    Permission.evaluate("skill", skill.name, agent.permission).action !== "deny"
  )
})
```

### Skill 在 System Prompt 中的呈现

```typescript
// packages/opencode/src/skill/index.ts

export function fmt(list: Info[], opts: { verbose: boolean }) {
  if (opts.verbose) {
    return [
      "<available_skills>",
      ...list.flatMap(skill => [
        "  <skill>",
        `    <name>${skill.name}</name>`,
        `    <description>${skill.description}</description>`,
        `    <location>${pathToFileURL(skill.location).href}</location>`,
        "  </skill>",
      ]),
      "</available_skills>",
    ].join("\n")
  }
  return ["## Available Skills", ...list.map(s => `- **${s.name}**: ${s.description}`)].join("\n")
}
```

### Skill Discovery — 远程拉取

```typescript
// packages/opencode/src/skill/discovery.ts

// 支持从 URL 拉取 skill 包
// 下载 → 解压 → 扫描 SKILL.md
```

### SkillTool — 运行时加载

```typescript
// packages/opencode/src/tool/skill.ts

// skill 工具让 Agent 在运行时决定加载哪个 skill
// Agent 根据任务描述选择合适的 skill
// 加载后，skill 的 content 被注入到上下文中
```

## 关键设计

### 兼容生态

OpenCode 扫描 `.claude/` 和 `.agents/` 目录，这意味着：
- Claude Code 的 skill 文件可以直接在 OpenCode 中使用
- 社区共享的 skill 不需要修改

### Markdown 即指令

Skill 的核心就是一段 Markdown 文本。没有特殊 DSL，没有编程接口。LLM 直接理解自然语言指令。

### 按需加载

Skill 不是全部注入 system prompt（那样会浪费 token），而是：
1. 列表信息（name + description）在 system prompt 中
2. Agent 根据需要通过 `skill` 工具加载具体内容

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 格式 | Markdown + YAML frontmatter | Markdown |
| 位置 | 多路径 + 兼容 `.claude/` | `.claude/` |
| 远程拉取 | `skills.urls` 配置 | 无 |
| 权限控制 | Agent 级别过滤 | 无 |
| 加载方式 | 按需 (skill 工具) | 全量注入 |
| 社区生态 | 互相兼容 | 自有生态 |

---

[← S06 提示词工程](s06-prompt-engineering.md) | [S08 MCP 集成 →](s08-mcp-integration.md)
