# 第七章：提示词组装 —— 系统提示与 Skill

> **格言**：好的提示词是分层的，像洋葱一样。

## 上回说到

压缩机制确保上下文不会溢出。现在让我们回到 LLM 调用的输入端，看看系统提示是如何组装的。

## 代码路径

### 1. 系统提示的三层结构

```typescript
// src/session/prompt.ts（loop 内部）
const skills = await SystemPrompt.skills(agent)
const system = [
  ...(await SystemPrompt.environment(model)),  // 层1: 环境信息
  ...(skills ? [skills] : []),                  // 层2: Skill 列表
  ...(await InstructionPrompt.system()),        // 层3: .opencode/instructions.md
]
```

### 2. 层1：Provider 系统提示 + 环境信息

```typescript
// src/session/system.ts
export function provider(model: Provider.Model) {
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  if (model.api.id.includes("gpt")) return [PROMPT_GPT]
  if (model.api.id.includes("gemini")) return [PROMPT_GEMINI]
  return [PROMPT_DEFAULT]
}

export async function environment(model: Provider.Model) {
  return [`You are powered by the model named ${model.api.id}.
Here is useful information about the environment:
<env>
  Working directory: ${Instance.directory}
  Workspace root: ${Instance.worktree}
  Is git repo: ${project.vcs === "git" ? "yes" : "no"}
  Platform: ${process.platform}
  Today's date: ${new Date().toDateString()}
</env>`]
}
```

不同模型使用不同的基础提示词（`PROMPT_ANTHROPIC`、`PROMPT_GPT` 等），针对各自的行为特点优化。

### 3. 层2：Skill 列表

```typescript
// src/session/system.ts
export async function skills(agent: Agent.Info) {
  if (Permission.disabled(["skill"], agent.permission).has("skill")) return

  const list = await Skill.available(agent)
  return [
    "Skills provide specialized instructions for specific tasks.",
    "Use the skill tool to load a skill when a task matches its description.",
    Skill.fmt(list, { verbose: true }),
  ].join("\n")
}
```

Skill 发现过程（`src/skill/index.ts`）扫描多个位置：

```typescript
// src/skill/index.ts（简化）
// 扫描位置：
// 1. 项目目录: .opencode/{skill,skills}/**/SKILL.md
// 2. 外部目录: .claude/skills/**/SKILL.md, .agents/skills/**/SKILL.md
// 3. 全局目录: ~/.opencode/skills/**/SKILL.md
// 4. 配置文件中指定的目录
```

每个 `SKILL.md` 使用 frontmatter 描述自己：

```markdown
---
name: my-skill
description: Does something useful
---

# Skill Instructions

When this skill is loaded, follow these steps...
```

Skill 只在系统提示中**列出名称和描述**。只有当 LLM 调用 `skill` 工具时，完整内容才被加载。

### 4. 层3：指令文件

`InstructionPrompt` 加载项目级的自定义指令：

```
.opencode/instructions.md    → 项目级指令
~/.opencode/instructions.md  → 全局指令
```

### 5. 在 LLM 调用中的位置

```typescript
// src/session/llm.ts:L72（stream 函数内部）
const system: string[] = []
system.push(
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,             // 上面组装的三层提示
    ...(input.user.system ? [input.user.system] : []),  // 用户消息附带的 system
  ].filter(x => x).join("\n"),
)

// Plugin 可以修改系统提示
await Plugin.trigger("experimental.chat.system.transform", { model }, { system })

// 发送给 LLM
const messages = [
  ...system.map((x) => ({ role: "system", content: x })),
  ...input.messages,  // 对话历史
]
```

系统提示被组装为两个 system message（利用 Anthropic 的缓存机制）：
1. 第一个 system message：基础提示 + 环境 + skills（相对稳定，可缓存）
2. 第二个 system message：指令文件等（可能变化）

### 6. Agent 自定义提示

某些 Agent 有自己的提示词，**替换**而非追加默认提示：

```typescript
// src/session/llm.ts:L74
input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)
```

例如 `compaction` agent 使用 `PROMPT_COMPACTION`，`explore` agent 使用 `PROMPT_EXPLORE`。

## 架构图

![ch07-prompts](../images/ch07-prompts.webp)

## 关键洞察

1. **提示词分层组装**：provider 基础 → 环境信息 → skills → 指令文件 → agent 覆盖
2. **Skill 是懒加载的**：系统提示只包含列表，调用 `skill` 工具才加载完整内容
3. **Plugin 可拦截修改**：`experimental.chat.system.transform` 钩子让插件修改最终提示
4. **缓存友好**：稳定部分放前面，变化部分放后面，利用 Anthropic 的 prompt caching

## 下一章预告

内置工具之外，OpenCode 还支持通过 MCP 连接外部工具服务器，以及通过 LSP 获取代码智能。

---

← [上一章：第六章：上下文管理](./ch06-compaction.md) | [下一章：第八章：外部集成](./ch08-mcp-lsp.md) →
