# S06 — 提示词工程：每个模型一套人格

> **格言：** "每个模型一套人格"

[← S05 权限系统](s05-permission-system.md) | [S07 Skill 系统 →](s07-skill-system.md)

---

## 问题

不同模型有不同的特性（Claude 善于长文本推理，GPT 善于结构化输出，Gemini 有大上下文窗口）。如何为每个模型定制最佳的系统提示词？

## 架构概览

```
┌────────────────────────────────────────────────┐
│           Prompt 系统                           │
│                                                │
│  session/prompt/         agent/prompt/          │
│  ├── anthropic.txt       ├── compaction.txt     │
│  ├── beast.txt           ├── explore.txt        │
│  ├── gemini.txt          ├── summary.txt        │
│  ├── gpt.txt             └── title.txt          │
│  ├── codex.txt                                  │
│  ├── default.txt                                │
│  └── trinity.txt                                │
│                                                │
│  SystemPrompt.provider(model) → 选择基础 prompt │
│  SystemPrompt.environment()   → 注入环境信息    │
│  SystemPrompt.skills()        → 注入可用技能    │
│                                                │
│  最终 system prompt =                           │
│    agent.prompt || provider prompt              │
│    + custom system from user message            │
│    + Plugin transforms                          │
└────────────────────────────────────────────────┘
```

## 核心代码

### 模型到 Prompt 的映射

```typescript
// packages/opencode/src/session/system.ts

export namespace SystemPrompt {
  export function provider(model: Provider.Model) {
    if (model.api.id.includes("gpt-4") || model.api.id.includes("o1") || model.api.id.includes("o3"))
      return [PROMPT_BEAST]          // GPT-4/o1/o3 → beast prompt
    if (model.api.id.includes("gpt")) {
      if (model.api.id.includes("codex")) return [PROMPT_CODEX]
      return [PROMPT_GPT]            // GPT-5 等 → gpt prompt
    }
    if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
    if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
    if (model.api.id.toLowerCase().includes("trinity")) return [PROMPT_TRINITY]
    return [PROMPT_DEFAULT]          // 其他 → 通用 prompt
  }
}
```

### 环境信息注入

```typescript
// packages/opencode/src/session/system.ts

export async function environment(model: Provider.Model) {
  const project = Instance.project
  return [
    [
      `You are powered by the model named ${model.api.id}.`,
      `Here is some useful information about the environment:`,
      `<env>`,
      `  Working directory: ${Instance.directory}`,
      `  Workspace root folder: ${Instance.worktree}`,
      `  Is directory a git repo: ${project.vcs === "git" ? "yes" : "no"}`,
      `  Platform: ${process.platform}`,
      `  Today's date: ${new Date().toDateString()}`,
      `</env>`,
    ].join("\n"),
  ]
}
```

### 技能信息注入

```typescript
// packages/opencode/src/session/system.ts

export async function skills(agent: Agent.Info) {
  if (Permission.disabled(["skill"], agent.permission).has("skill")) return

  const list = await Skill.available(agent)
  return [
    "Skills provide specialized instructions and workflows.",
    "Use the skill tool to load a skill when a task matches.",
    Skill.fmt(list, { verbose: true }),
  ].join("\n")
}
```

### Anthropic Prompt 特点

```
// packages/opencode/src/session/prompt/anthropic.txt (摘要)

You are OpenCode, the best coding agent on the planet.
You are an interactive CLI tool that helps users with software engineering tasks.

# Tone and style
- Only use emojis if the user explicitly requests it.
- Your output will be displayed on a command line interface.
- Output text to communicate; never use tools as means to communicate.

# Professional objectivity
Prioritize technical accuracy over validating the user's beliefs.
Disagree when necessary, even if it may not be what the user wants to hear.

# Task Management
Use TodoWrite tools VERY frequently to track tasks.
Mark todos as completed as soon as you are done.
```

### Prompt 在 LLM 层的组装

```typescript
// packages/opencode/src/session/llm.ts

const system: string[] = []
system.push(
  [
    // 1. Agent 自定义 prompt 或 provider prompt
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    // 2. 调用方传入的额外 system prompt
    ...input.system,
    // 3. 用户消息附带的 system prompt
    ...(input.user.system ? [input.user.system] : []),
  ].filter(x => x).join("\n"),
)

// Plugin 可以 transform system prompt
await Plugin.trigger("experimental.chat.system.transform", { model: input.model }, { system })

// OpenAI OAuth → 用 instructions 而不是 system message
if (isOpenaiOauth) {
  options.instructions = system.join("\n")
}

// 普通模式 → system message + user messages
const messages = isOpenaiOauth ? input.messages : [
  ...system.map(x => ({ role: "system", content: x })),
  ...input.messages,
]
```

### 缓存优化

系统 prompt 分两部分以利用 prompt caching：

```typescript
// packages/opencode/src/session/llm.ts

// 如果 plugin transform 后 header 没变，保持两部分结构
// 第一部分（不变的 header）→ 可被缓存
// 第二部分（动态内容）→ 每次变化
if (system.length > 2 && system[0] === header) {
  const rest = system.slice(1)
  system.length = 0
  system.push(header, rest.join("\n"))
}
```

## 专用 Agent Prompt

### Explore Agent

```
// packages/opencode/src/agent/prompt/explore.txt

专注于代码探索：
- 使用 grep/glob 搜索
- 不修改任何文件
- 根据 thoroughness 级别调整搜索深度
```

### Compaction Agent

```
// packages/opencode/src/agent/prompt/compaction.txt

总结对话历史，保留关键上下文。
```

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 模型适配 | 每个模型族独立 prompt | 单一 Claude prompt |
| Prompt 存储 | `.txt` 文件，可替换 | 硬编码 |
| 环境注入 | 动态 `<env>` 标签 | 类似 |
| Plugin 扩展 | `system.transform` hook | 无 |
| 缓存优化 | 两部分结构 | Anthropic API 缓存 |
| 自定义 | `agent.prompt` 配置项 | 不支持 |

---

[← S05 权限系统](s05-permission-system.md) | [S07 Skill 系统 →](s07-skill-system.md)
