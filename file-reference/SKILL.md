---
name: skill-file-reference
description: Reference for creating Claude Code skill files. Use when asked how to write a SKILL.md, what fields are available, or how to structure a custom slash command.
disable-model-invocation: true
allowed-tools: Read Write Glob
tags: [local]
---

# Claude Code Skill File Reference

A skill file is a `SKILL.md` with two parts: YAML frontmatter and a markdown body.

## 1. YAML Frontmatter (between `---` markers)

| Field | Purpose |
|-------|---------|
| `name` | Slash command name (lowercase, hyphens, max 64 chars). Defaults to directory name. |
| `description` | When to use it — Claude uses this for auto-invocation (250 char limit). Front-load the key use case. |
| `argument-hint` | Autocomplete hint, e.g. `[issue-number]` or `[filename] [format]` |
| `allowed-tools` | Space-separated tools allowed without permission (e.g. `Read Grep Glob`) |
| `model` | Model override when skill is active |
| `effort` | `low`, `medium`, `high`, or `max` |
| `context` | Set to `fork` to run in isolated subagent context |
| `agent` | Subagent type with `context: fork` (e.g. `Explore`, `Plan`, `general-purpose`) |
| `disable-model-invocation` | `true` = only manual `/name` trigger, Claude won't auto-invoke |
| `user-invocable` | `false` = hidden from `/` menu, only Claude can invoke it |
| `paths` | Glob patterns limiting when skill auto-activates; comma-separated string or YAML list |
| `shell` | `bash` (default) or `powershell` for inline commands |
| `hooks` | Hooks scoped to this skill's lifecycle |

## 2. Markdown Body (the instructions)

The actual prompt Claude follows when the skill is invoked. Supports dynamic substitutions:

- `$ARGUMENTS` — all arguments passed to the skill
- `$ARGUMENTS[N]` or `$N` — specific argument by index (0-based)
- `${CLAUDE_SESSION_ID}` — current session ID
- `${CLAUDE_SKILL_DIR}` — directory containing this SKILL.md

**Shell command injection** (runs before Claude sees content):
- `` !`command` `` — inline shell execution
- ` ```! ` — multi-line shell blocks

## 3. File Locations

- **Personal** (all projects): `~/.claude/skills/<name>/SKILL.md`
- **Project** (repo-scoped): `.claude/skills/<name>/SKILL.md`

## 4. Supporting Files

Skills can include additional files alongside SKILL.md:

```
my-skill/
├── SKILL.md           # Main instructions (required)
├── reference.md       # Reference material
├── examples.md        # Usage examples
└── scripts/
    └── helper.py      # Utility scripts
```

Reference supporting files from SKILL.md so Claude knows when to load them.

## Minimal Example

```yaml
---
name: explain-code
description: Explains code with diagrams and analogies
allowed-tools: Read Grep Glob
---

When explaining code, include:
1. A relatable analogy
2. An ASCII diagram showing flow
3. Step-by-step walkthrough
```