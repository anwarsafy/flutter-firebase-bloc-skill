# How to Install & Use This Skill Anywhere

Copy the `flutter-firebase-bloc/` folder (containing `SKILL.md`, `references/`, and `templates/`) into your preferred tool:

| IDE / Agent | Where to Place It | Notes |
|---|---|---|
| **Claude Code** | `~/.claude/skills/flutter-firebase-bloc/` (global) or `.claude/skills/flutter-firebase-bloc/` (project) | Automatically discovered via `SKILL.md` frontmatter |
| **Cursor** | `.cursor/skills/flutter-firebase-bloc/` (or copy `SKILL.md` into `.cursor/rules/flutter-firebase.mdc`) | Supports full `references/` progressive loading |
| **Windsurf** | `.windsurf/skills/flutter-firebase-bloc/` (or `.windsurfrules`) | Point rules to read `references/*.md` on demand |
| **GitHub Copilot (VS Code)** | `.github/copilot-instructions.md` (paste `SKILL.md`) or `.github/skills/flutter-firebase-bloc/` | Keep `references/` in `.github/skills/flutter-firebase-bloc/references/` |
| **OpenAI Codex CLI** | `~/.codex/skills/flutter-firebase-bloc/` or project `AGENTS.md` | Standard Agent Skills folder layout |
| **Gemini CLI / Antigravity / AI Studio** | `~/.gemini/skills/flutter-firebase-bloc/` or `.agent/skills/flutter-firebase-bloc/` | `SKILL.md` + `references/` + `templates/` |
| **ChatGPT / Claude Web / Gemini Gem** | Upload `SKILL.md` + `references/*.md` as Knowledge files, and paste `SKILL.md` into System Instructions | Works in any chat UI without an IDE |
