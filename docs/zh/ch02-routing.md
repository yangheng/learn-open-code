# 第二章：路由 —— Agent 与 Provider 的选择

> **格言**：选对了模型，就赢了一半。

## 上回说到

用户消息已创建，绑定了 `agent: "build"` 和 `model: { providerID: "anthropic", modelID: "claude-sonnet-4" }`。`SessionPrompt.loop()` 被调用，即将开始处理。

## 代码路径

### 1. loop() 的开头：读取历史消息

```typescript
// src/session/prompt.ts:L103（loop 函数）
while (true) {
  let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))
  
  let lastUser: MessageV2.User | undefined
  let lastAssistant: MessageV2.Assistant | undefined
  for (let i = msgs.length - 1; i >= 0; i--) {
    const msg = msgs[i]
    if (!lastUser && msg.info.role === "user") lastUser = msg.info as MessageV2.User
    if (!lastAssistant && msg.info.role === "assistant") lastAssistant = msg.info as MessageV2.Assistant
  }
```

Loop 做的第一件事是从数据库逆序扫描所有消息，找到**最后一条用户消息**——这里面包含了 agent 和 model 的选择。

### 2. 解析 Agent

```typescript
// src/session/prompt.ts（loop 内部）
const agent = await Agent.get(lastUser.agent)
```

Agent 定义在 `src/agent/agent.ts` 中，是一个**静态配置对象**，不是运行时实例：

```typescript
// src/agent/agent.ts:L65
const agents: Record<string, Info> = {
  build: {
    name: "build",
    description: "The default agent. Executes tools based on configured permissions.",
    permission: Permission.merge(defaults, Permission.fromConfig({ question: "allow", plan_enter: "allow" }), user),
    mode: "primary",
    native: true,
  },
  plan: {
    name: "plan",
    description: "Plan mode. Disallows all edit tools.",
    permission: Permission.merge(defaults, Permission.fromConfig({ edit: { "*": "deny" } }), user),
    mode: "primary",
    native: true,
  },
  explore: {
    name: "explore",
    permission: Permission.merge(defaults, Permission.fromConfig({ "*": "deny", grep: "allow", read: "allow" }), user),
    mode: "subagent",
    native: true,
  },
  // ...
}
```

每个 Agent 的核心属性：
- **permission**：一套 Ruleset，决定该 Agent 能用哪些工具
- **mode**：`primary`（用户直接选择）或 `subagent`（由 task tool 调用）
- **prompt**：可选的自定义系统提示词
- **model**：可选的绑定模型

### 3. 解析 Model

```typescript
// src/session/prompt.ts（loop 内部）
const model = await Provider.getModel(lastUser.model.providerID, lastUser.model.modelID)
```

Provider 层（`src/provider/provider.ts`）是 OpenCode 最复杂的模块之一。它要做三件事：

**a) 发现可用的 Provider**

Provider 从多个来源聚合：
```typescript
// src/provider/provider.ts（state 函数内部）
// 1. models.dev 数据库（内置模型清单）
const modelsDev = await ModelsDev.get()
const database = mapValues(modelsDev, fromModelsDevProvider)

// 2. 环境变量（如 ANTHROPIC_API_KEY）
for (const [id, provider] of Object.entries(database)) {
  const apiKey = provider.env.map((item) => env[item]).find(Boolean)
  if (apiKey) mergeProvider(providerID, { source: "env", key: apiKey })
}

// 3. Auth 存储（opencode auth 命令保存的凭证）
for (const [id, provider] of Object.entries(await Auth.all())) {
  mergeProvider(providerID, { source: "api", key: provider.key })
}

// 4. 用户配置（opencode.json 中的 provider 字段）
for (const [id, provider] of configProviders) {
  mergeProvider(providerID, { source: "config" })
}
```

**b) 加载 SDK**

每个 Provider 需要一个 AI SDK 实现。OpenCode 内置了 20+ 个：

```typescript
// src/provider/provider.ts:L64
const BUNDLED_PROVIDERS: Record<string, (options: any) => BundledSDK> = {
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  // ... 20+ more
}
```

**c) 获取 Language Model**

```typescript
// src/provider/provider.ts
export async function getLanguage(model: Model): Promise<LanguageModelV3> {
  const sdk = await getSDK(model)
  const language = s.modelLoaders[model.providerID]
    ? await s.modelLoaders[model.providerID](sdk, model.api.id, options)
    : sdk.languageModel(model.api.id)
  return language
}
```

### 4. 默认模型选择

如果用户没指定模型，OpenCode 使用智能回退：

```typescript
// src/provider/provider.ts
export async function defaultModel() {
  const cfg = await Config.get()
  if (cfg.model) return parseModel(cfg.model)

  // 检查最近使用的模型
  const recent = await Filesystem.readJson(path.join(Global.Path.state, "model.json"))
  for (const entry of recent) {
    if (providers[entry.providerID]?.models[entry.modelID]) return entry
  }

  // 回退到第一个可用 provider 的最佳模型
  const provider = Object.values(providers).find(...)
  const [model] = sort(Object.values(provider.models))  // 按 gpt-5 > claude-sonnet-4 > gemini 排序
  return { providerID: provider.id, modelID: model.id }
}
```

## 架构图

![ch02-routing](../images/ch02-routing.webp)

## 关键洞察

1. **Agent 是权限+提示的容器**：它不运行代码，只是一组配置，决定了 LLM 能调用哪些工具
2. **Provider 聚合四种来源**：models.dev 数据库 + 环境变量 + auth 存储 + 用户配置
3. **模型选择有优先级链**：用户指定 > 配置文件 > 最近使用 > 自动检测
4. **Agent 的 permission 是白名单机制**：`build` 什么都能做，`plan` 不能编辑，`explore` 只能读

## 下一章预告

Agent 和 Model 都已确定。现在一切就绪，可以进入核心循环了——消息发给 LLM，LLM 回复文本或工具调用，循环往复直到完成。

---

← [上一章：第一章：入口](./ch01-entry-point.md) | [下一章：第三章：核心循环](./ch03-core-loop.md) →
