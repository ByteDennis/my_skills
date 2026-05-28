---
id: 6a1660c455a74
name: Codex
tags: [codex, cli, openai, keymap]
updated_at: 2026-05-27T04:53:28.773961Z
---

OpenAI's terminal-based coding agent. `codex` reads, edits, and runs code locally with approval controls.

## Login

```bash
codex login
codex login --device-auth
printenv OPENAI_API_KEY | codex login --with-api-key
```

> [!WARNING]
> If `OPENAI_API_KEY` is set, Codex may use it instead of stored login credentials.

## Core Commands

| Command | Purpose |
|---|---|
| `codex` | Launch the interactive CLI |
| `codex exec` | Run non-interactively; supports `--json` |
| `codex review` | Run a code review non-interactively |
| `codex resume` | Resume a previous session |
| `codex cloud exec` | Submit a Codex Cloud task |
| `codex mcp add` | Add an MCP server |
| `codex features list` | Inspect feature flags |
| `codex doctor` | Diagnose the local installation |

## Input Prefixes

| Prefix | Action |
|---|---|
| `@` | Fuzzy file or directory search and attach to the conversation |
| `!` | Execute a shell command directly |
| `/` | Open the slash-command menu |

## Approval

These are the current CLI flag names.

| Flag | Values |
|---|---|
| `--ask-for-approval` | `untrusted`, `on-request`, `never` |
| `--sandbox` | `read-only`, `workspace-write`, `danger-full-access` |

Emergency bypass:

```bash
--dangerously-bypass-approvals-and-sandbox
```

## Common Flags

| Flag | Use |
|---|---|
| `--model` | Choose the model |
| `--image` | Attach one or more images |
| `--profile` | Layer `$CODEX_HOME/<name>.config.toml` on top of the base user config |
| `--add-dir` | Add extra writable directories |
| `--oss` | Use the open-source provider |
| `--local-provider` | Pick `lmstudio` or `ollama` |
| `--search` | Enable live web search |
| `--json` | Print JSONL output for `codex exec` |
| `--output-last-message` | Write the final assistant message to a file |
| `--strict-config` | Fail on unknown config keys |

## User Needs To Know

| Topic | Why it matters |
|---|---|
| `codex resume` | Continue a prior session instead of starting over. |
| `codex exec` | Best for scripted, repeatable, non-interactive work. |
| `codex review` | Use for code review, not just editing. |
| `AGENTS.md` | Put repo-specific rules here so Codex inherits them. |
| `--sandbox` and `--ask-for-approval` | Tune how autonomous Codex should be before you let it act. |
| `codex sandbox` | Good for reproducing a command or testing a change in isolation. |
| `OPENAI_API_KEY` | If set, it can override stored login credentials. |
| `/keymap` | Inspect and remap interactive shortcuts from inside Codex. |

## Configuration

```toml
# ~/.codex/config.toml
model = "gpt-5.4"
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[tui]
vim_mode_default = true
```

| Setting | Notes |
|---|---|
| `-c key=value` | Override a config value from the command line |
| `--profile NAME` | Layer `$CODEX_HOME/NAME.config.toml` on top of the base user config |
| `--strict-config` | Catch stale config fields early |

## Claude Code Migration

| Goal | Suggested setup |
|---|---|
| Similar editing feel | Set `vim_mode_default = true` and keep the prompt in the editor flow you already know. |
| Similar shortcuts | Open `/keymap` and map Codex actions to your preferred keys. |
| `Ctrl+O` muscle memory | Bind the history/recent-items action to `Ctrl+O` if your Codex build exposes that action and your terminal does not intercept the key first. |
| `Ctrl+E` muscle memory | Bind the command/menu action to `Ctrl+E` the same way. |
| External editor flow | `Ctrl+G` opens the prompt in an external editor. |

> [!NOTE]
> If `Ctrl+O` or `Ctrl+E` are already used by your terminal, shell, or editor, disable those bindings first. Then mirror the same keys in `/keymap` so Codex receives them.

## Project Instructions

Keep repo-specific instructions in [`AGENTS.md`](./AGENTS.md) at the project root.

## Useful Commands

| Command | Notes |
|---|---|
| `codex login status` | Show login status |
| `codex mcp list` | List configured MCP servers |
| `codex features list` | Show feature stage and effective state |
| `codex update` | Update Codex |
| `codex sandbox` | Run a command inside a Codex sandbox |

## Notes

| Topic | Current state |
|---|---|
| Project instructions | `AGENTS.md` is the project instruction file Codex reads from the repo root downward |
| Cloud tasks | `codex cloud` is experimental and includes `exec`, `status`, `list`, `apply`, and `diff` |
| Keymap | Use `/keymap` for interactive remapping; `Ctrl+G` opens the prompt in an external editor |
