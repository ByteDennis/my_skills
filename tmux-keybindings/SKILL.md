---
name: tmux-keybindings
description: Tmux keybindings quick reference (prefix = C-a)
tags: [tmux, keybindings, terminal, local]
---

# Tmux Keybindings (Prefix: C-a)

## Pane Management

| Key | Action |
|-----|--------|
| `C-a \|` | Split horizontal |
| `C-a -` | Split vertical |
| `C-a h/j/k/l` | Navigate panes (vim style) |
| `M-h/j/k/l` | Navigate panes (no prefix) |
| `C-a H/J/K/L` | Resize pane (repeatable) |
| `C-a p` | Kill pane |
| `C-a b` | Break pane to own window |
| `C-a S` | Toggle synchronized panes |

## Window Management

| Key | Action |
|-----|--------|
| `C-a c` | New window (prompts for name) |
| `C-a C-h` | Previous window |
| `C-a C-l` | Next window |
| `C-a <` | Move window left |
| `C-a >` | Move window right |
| `C-a r` | Rename window |
| `C-a x` | Kill window (confirm) |

## Session Management

| Key | Action |
|-----|--------|
| `C-a R` | Rename session |
| `C-a w` | Choose session/window picker |
| `C-a N` | Next session |
| `C-a B` | Previous session |
| `C-a X` | Kill session (confirm) |

## Copy Mode (vi-style)

| Key | Action |
|-----|--------|
| `C-a [` | Enter copy mode |
| `v` | Begin selection |
| `C-v` | Block select |
| `y` | Yank to clipboard |
| `C-a P` | Paste buffer |

## Plugins (TPM)

| Key | Action |
|-----|--------|
| `C-a I` | Install plugins |
| `C-a U` | Update plugins |
| `C-a C-s` | Save session (resurrect) |
| `C-a C-r` | Restore session |
| `C-a M-r` | Reload config |