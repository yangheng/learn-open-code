# S06 — Prompt Engineering: One Persona Per Model

> **Motto:** "One persona per model"

[← S05 Permission System](s05-permission-system.md) | [S07 Skill System →](s07-skill-system.md)

---

OpenCode maintains separate system prompts for each model family: `anthropic.txt`, `beast.txt` (GPT-4/o1/o3), `gpt.txt`, `gemini.txt`, `codex.txt`, `default.txt`, `trinity.txt`. `SystemPrompt.provider(model)` selects the right prompt based on model ID. Environment info (working directory, platform, date) and available skills are injected dynamically. Plugins can transform the system prompt via `experimental.chat.system.transform` hook. The prompt is split into two parts for cache optimization.

See [Chinese version](../zh/s06-prompt-engineering.md) for full code analysis.

---

[← S05 Permission System](s05-permission-system.md) | [S07 Skill System →](s07-skill-system.md)
