---
name: zsh-shell-utils
tags: [zsh, shell, aliases, functions, local]
---

# Zsh Shell Utilities

## Custom Functions

### `killport <port>` — Kill process on a port
```bash
killport 8080
```

### `extract <file>` — Extract any archive format
Supports: .tar.gz, .tar.bz2, .tar.xz, .zip, .7z, .rar, .zst
```bash
extract dataset.tar.gz
```

### `dsize [dir]` — Directory size summary (top 20)
Uses `dust` if available, falls back to `du`.
```bash
dsize /data
```

### `findlarge [size]` — Find files larger than size (default 100M)
Uses `fd` if available.
```bash
findlarge 500M
```

### `psg <pattern>` — Process grep
Uses `procs` if available.
```bash
psg python
```

## Key Aliases

| Alias | Expands To |
|-------|-----------|
| `ll` | `eza -lah --icons --git` |
| `lt` | `eza -lah --icons --sort=modified` |
| `tree` | `eza --tree --icons --level=3` |
| `gs` | `git status` |
| `gl` | `git log --oneline --graph -20` |
| `dps` | `docker ps` (formatted) |
| `lzd` | `lazydocker` |
| `gpu` | `nvidia-smi` |
| `gpuw` | `watch -n 1 nvidia-smi` |
| `top` | `btop` |
| `df` | `duf` |
| `nv` | `nvim` |
| `py` | `python` |
| `mm` | `micromamba` |
| `pip` | `uv pip` |

## Shell Config Shortcuts

| Alias | What |
|-------|------|
| `reload` | `source ~/.zshrc` |
| `zalias` | Edit aliases + reload |
| `zrc` | Edit zshrc + reload |
| `zex` | Edit exports + reload |