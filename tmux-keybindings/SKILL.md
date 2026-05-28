---
id: skill-tmux-keybindings
name: Tmux Keyboard
tags: [tmux, keybindings, terminal, local]
updated_at: 2026-05-28T08:34:37.815959Z
---

# Tmux Shortcuts — dalab2

**Prefix:** `C-a` (Ctrl-a). Double-tap `C-a` to send a literal `C-a` to the program.

In tables below, `<P>` = press prefix first.

---

## Config & Help

| Keys | Action |
|---|---|
| `<P> C-r` | Reload `~/.tmux.conf` |
| `<P> r` | Rename current window |
| `<P> R` | Rename current session |

## Windows

| Keys | Action |
|---|---|
| `<P> c` | New window (in current pane's directory) |
| `<P> C-h` | Previous window |
| `<P> C-l` | Next window |
| `<P> H` | Move current window **left** (swap with previous) |
| `<P> L` | Move current window **right** (swap with next) |
| `<P> <` | Move current window left (alt) |
| `<P> >` | Move current window right (alt) |
| `<P> x` | Kill window (with confirmation) |

## Panes — Splits

| Keys | Action |
|---|---|
| `<P> \|` | Split horizontally (new pane to the right) |
| `<P> -` | Split vertically (new pane below) |
| `<P> p` | Kill current pane |
| `<P> b` | Break current pane into its own window |

## Panes — Navigation (vim hjkl)

| Keys | Action |
|---|---|
| `<P> h` / `j` / `k` / `l` | Focus pane left / down / up / right |
| `M-h` / `M-j` / `M-k` / `M-l` | Same, **no prefix needed** (Alt+hjkl) |

## Panes — Resize

| Keys | Action |
|---|---|
| `<P> J` | Resize pane down 5 (repeatable) |
| `<P> K` | Resize pane up 5 (repeatable) |

> Note: `H` and `L` are reserved for window swap. Resize left/right is unbound — use mouse drag on pane borders.

## Sessions

| Keys | Action |
|---|---|
| `<P> w` | Visual session/window picker |
| `<P> N` | Switch to next session |
| `<P> B` | Switch to previous session (Back) |
| `<P> X` | Kill current session (with confirmation) |

## Copy Mode (vi-style)

Enter copy mode: `<P> [` &nbsp;&nbsp; · &nbsp;&nbsp; Exit: `Esc` or `q`

| Keys | Action |
|---|---|
| `h` / `j` / `k` / `l` | Move cursor |
| `v` | Begin visual selection |
| `C-v` | Toggle block (rectangle) selection |
| `y` | Yank selection to system clipboard (xclip) |
| `Esc` | Cancel / exit copy mode |
| Mouse drag | Select and copy to clipboard on release |
| `<P> P` | Paste from tmux buffer |

## Utilities

| Keys | Action |
|---|---|
| `<P> S` | Toggle synchronize-panes (broadcast input to all panes) |
| `<P> I` | Install TPM plugins |
| `<P> U` | Update TPM plugins |
| `<P> M-u` | Uninstall removed TPM plugins |
| `<P> C-s` | tmux-resurrect: manual save |
| `<P> C-r` | **Overridden** to reload config (resurrect restore is unbound) |

## Mouse

- Mouse is **on**: click to focus pane, scroll to navigate, drag borders to resize, drag in pane to select+copy.

---

_Generated from `~/.tmux.conf`._
