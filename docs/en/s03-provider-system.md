# S03 — Provider System: 20+ Providers, One `streamText()`

> **Motto:** "20+ providers, one `streamText()`"

[← S02 Tool System](s02-tool-system.md) | [S04 Session Management →](s04-session-management.md)

---

OpenCode bundles 20+ AI SDK providers (Anthropic, OpenAI, Google, Azure, Bedrock, xAI, Mistral, Groq, Copilot, OpenRouter, etc.) in a single `BUNDLED_PROVIDERS` map. Custom loaders handle provider-specific quirks (OpenAI uses `responses()` API, Anthropic needs beta headers). `ProviderTransform` adapts temperature, max tokens, and provider options per model. An SSE timeout wrapper (`wrapSSE`) protects against stalled streams. For LiteLLM proxies, a dummy `_noop` tool is injected when history contains tool calls but no tools are active.

See [Chinese version](../zh/s03-provider-system.md) for full code analysis.

---

[← S02 Tool System](s02-tool-system.md) | [S04 Session Management →](s04-session-management.md)
