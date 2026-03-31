# S03 — Provider 系统：20+ 供应商，一个 `streamText()`

> **格言：** "20+ 供应商，一个 `streamText()`"

[← S02 工具系统](s02-tool-system.md) | [S04 会话管理 →](s04-session-management.md)

---

## 问题

不同 LLM 供应商有不同的 API、认证方式、模型能力。如何让 Agent 循环对此无感知，用同一段代码调用 Claude、GPT、Gemini、Copilot？

## 架构概览

```
┌─────────────────────────────────────────────────┐
│                 Provider 系统                    │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │          BUNDLED_PROVIDERS                │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  │  │
│  │  │anthropic │  │ openai   │  │ google │  │  │
│  │  ├──────────┤  ├──────────┤  ├────────┤  │  │
│  │  │ azure    │  │ bedrock  │  │ vertex │  │  │
│  │  ├──────────┤  ├──────────┤  ├────────┤  │  │
│  │  │  xai     │  │ mistral  │  │  groq  │  │  │
│  │  ├──────────┤  ├──────────┤  ├────────┤  │  │
│  │  │copilot   │  │openrouter│  │ cohere │  │  │
│  │  ├──────────┤  ├──────────┤  ├────────┤  │  │
│  │  │ vercel   │  │ gitlab   │  │deepinfr│  │  │
│  │  └──────────┘  └──────────┘  └────────┘  │  │
│  └───────────────────────────────────────────┘  │
│                      │                          │
│                      ▼                          │
│  ┌───────────────────────────────────────────┐  │
│  │     Provider.getLanguage(model)           │  │
│  │     → LanguageModelV3                     │  │
│  └───────────────────┬───────────────────────┘  │
│                      │                          │
│                      ▼                          │
│  ┌───────────────────────────────────────────┐  │
│  │  ProviderTransform                        │  │
│  │  • options()       • maxOutputTokens()    │  │
│  │  • temperature()   • providerOptions()    │  │
│  │  • smallOptions()  • topP() / topK()      │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

## 核心代码

### 内置供应商注册表

OpenCode 直接 import 20+ 个 AI SDK provider：

```typescript
// packages/opencode/src/provider/provider.ts

import { createAnthropic } from "@ai-sdk/anthropic"
import { createAzure } from "@ai-sdk/azure"
import { createGoogleGenerativeAI } from "@ai-sdk/google"
import { createOpenAI } from "@ai-sdk/openai"
import { createOpenRouter } from "@openrouter/ai-sdk-provider"
import { createXai } from "@ai-sdk/xai"
import { createMistral } from "@ai-sdk/mistral"
import { createGroq } from "@ai-sdk/groq"
// ...更多

const BUNDLED_PROVIDERS: Record<string, (options: any) => BundledSDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openai-compatible": createOpenAICompatible,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  "@ai-sdk/deepinfra": createDeepInfra,
  "@ai-sdk/cerebras": createCerebras,
  "@ai-sdk/cohere": createCohere,
  "@ai-sdk/gateway": createGateway,
  "@ai-sdk/togetherai": createTogetherAI,
  "@ai-sdk/perplexity": createPerplexity,
  "@ai-sdk/vercel": createVercel,
  "gitlab-ai-provider": createGitLab,
  "@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,
}
```

### 自定义加载器 (CUSTOM_LOADERS)

某些供应商需要特殊处理：

```typescript
// packages/opencode/src/provider/provider.ts

const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  async anthropic() {
    return {
      autoload: false,
      options: {
        headers: {
          "anthropic-beta": "interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14",
        },
      },
    }
  },
  openai: async () => ({
    autoload: false,
    async getModel(sdk, modelID) {
      return sdk.responses(modelID)  // OpenAI 用 responses API
    },
  }),
  async opencode(input) {
    // OpenCode 自有模型：检查是否有 API key
    const hasKey = await (async () => { /* ... */ })()
    if (!hasKey) {
      // 没有 key → 只保留免费模型
      for (const [key, value] of Object.entries(input.models)) {
        if (value.cost.input === 0) continue
        delete input.models[key]
      }
    }
    return { autoload: Object.keys(input.models).length > 0 }
  },
}
```

### SSE 超时保护

OpenCode 对流式响应添加了 SSE 超时保护：

```typescript
// packages/opencode/src/provider/provider.ts

function wrapSSE(res: Response, ms: number, ctl: AbortController) {
  if (!res.headers.get("content-type")?.includes("text/event-stream")) return res
  const reader = res.body.getReader()
  const body = new ReadableStream({
    async pull(ctrl) {
      const part = await new Promise((resolve, reject) => {
        const id = setTimeout(() => {
          const err = new Error("SSE read timed out")
          ctl.abort(err)
          reject(err)
        }, ms)
        reader.read().then(part => { clearTimeout(id); resolve(part) },
                           err => { clearTimeout(id); reject(err) })
      })
      if (part.done) { ctrl.close(); return }
      ctrl.enqueue(part.value)
    },
  })
  return new Response(body, { headers: new Headers(res.headers), status: res.status })
}
```

### ProviderTransform — 模型能力适配

不同模型有不同的能力（温度控制、输出长度、thinking 等），`ProviderTransform` 统一处理：

```typescript
// packages/opencode/src/provider/transform.ts

export namespace ProviderTransform {
  export const OUTPUT_TOKEN_MAX = 128_000

  export function options(input: { model: Provider.Model; sessionID: string }) {
    // 根据模型能力返回不同选项
    // 例如 Anthropic 支持 extended thinking
    // OpenAI 支持 reasoning effort
  }

  export function maxOutputTokens(model: Provider.Model) {
    // 根据模型配置返回最大输出 token 数
  }

  export function temperature(model: Provider.Model) {
    // 某些模型不支持 temperature
  }
}
```

### Model 信息来源

模型元数据来自 `models.dev`：

```typescript
// packages/opencode/src/provider/schema.ts

export const ProviderID = {
  anthropic: "anthropic",
  openai: "openai",
  google: "google",
  opencode: "opencode",
  // ...
}
```

## LiteLLM 代理兼容

一个有趣的细节 —— 当检测到 LiteLLM 代理时，如果消息历史中有工具调用但当前没有工具，会注入一个 dummy 工具防止 API 报错：

```typescript
// packages/opencode/src/session/llm.ts

if (isLiteLLMProxy && Object.keys(tools).length === 0 && hasToolCalls(input.messages)) {
  tools["_noop"] = tool({
    description: "Do not call this tool. It exists only for API compatibility.",
    inputSchema: jsonSchema({ type: "object", properties: { reason: { type: "string" } } }),
    execute: async () => ({ output: "", title: "", metadata: {} }),
  })
}
```

## 与 Claude Code 的对比

| 方面 | OpenCode | Claude Code |
|------|----------|-------------|
| 供应商支持 | 20+ (任何 AI SDK 兼容) | 仅 Anthropic |
| SDK | Vercel AI SDK | 自有 API 客户端 |
| 模型切换 | 配置文件 / 运行时 | 不支持 |
| 认证方式 | env vars / OAuth / API key | API key |
| 流式保护 | SSE 超时 wrapSSE | 未知 |

---

[← S02 工具系统](s02-tool-system.md) | [S04 会话管理 →](s04-session-management.md)
