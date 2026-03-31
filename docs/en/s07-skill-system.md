# S07 — Skill System: Markdown is Capability

> **Motto:** "Markdown is capability"

[← S06 Prompt Engineering](s06-prompt-engineering.md) | [S08 MCP Integration →](s08-mcp-integration.md)

---

Skills are Markdown files with YAML frontmatter (`name`, `description`) scanned from multiple paths: `~/.claude/skills/`, `~/.agents/skills/` (Claude Code compatible), `.opencode/skills/`, configured paths, and remote URLs. Skills are loaded on-demand via the `skill` tool — only name/description appear in the system prompt, content is injected when the agent selects a skill. Permission filtering ensures agents only see skills they're allowed to use. `Discovery` module supports pulling skills from remote URLs.

See [Chinese version](../zh/s07-skill-system.md) for full code analysis.

---

[← S06 Prompt Engineering](s06-prompt-engineering.md) | [S08 MCP Integration →](s08-mcp-integration.md)
