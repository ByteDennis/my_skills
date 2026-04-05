---
name: fzf-search
tags: [fzf, search, shell, productivity, local]
---

# FZF Fuzzy Finder

## Shell Keybindings

| Key | Action |
|-----|--------|
| `Ctrl+T` | Fuzzy find files (with bat preview) |
| `Ctrl+R` | Fuzzy search history (via Atuin) |
| `Alt+C` | Fuzzy cd into directory (with eza preview) |

## Configuration

```bash
# Uses fd for file discovery (respects .gitignore)
FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'

# Layout: bottom 40%, reversed, with border
FZF_DEFAULT_OPTS="--height 40% --layout=reverse --border --info=inline"
```

## Common Patterns

### Find and edit a file
```bash
nvim $(fzf)
```

### Find and kill a process
```bash
kill -9 $(ps aux | fzf | awk '{print $2}')
```

### Git branch switcher
```bash
git checkout $(git branch | fzf | tr -d ' ')
```

### Docker container selector
```bash
docker exec -it $(docker ps --format '{{.Names}}' | fzf) bash
```