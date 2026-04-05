---
name: rsync-service
description: Rsync web UI service — file sync, profiles, filters, and .rsyncignore usage
tags: [rsync, sync, deployment, migration, local]
---

# Rsync Service

Web UI for rsync at `/rsync`. Dual-pane file browser with SSH remote browsing, profile management, and real-time transfer progress.

## Architecture

- **Backend**: `routes/rsync.py` — Flask blueprint, all endpoints under `/rsync/*`
- **Frontend**: `templates/rsync.html` — single-page with modals
- **Data**: `/data/rsync_profiles.json`, `/data/rsync_history.json`

## API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/rsync` | Serve UI |
| GET/POST/DELETE | `/rsync/profiles` | Profile CRUD |
| POST | `/rsync/browse` | Browse local/remote dirs |
| POST | `/rsync/preview` | Preview file contents |
| POST | `/rsync/sync` | Start sync job (async) |
| GET | `/rsync/status/<id>` | Poll job progress |
| POST | `/rsync/cancel/<id>` | Cancel running sync |
| GET/DELETE | `/rsync/history` | Sync history |

## Sync Options

| Option | Flag | Default |
|--------|------|---------|
| Compress | `-z` | on |
| Preserve timestamps | `--times` | on |
| Preserve permissions | `--perms` | off |
| Delete extra files | `--delete` | off |
| Dry-run | `--dry-run` | off |
| Read .rsyncignore | `--filter='dir-merge /.rsyncignore'` | on |
| Bandwidth limit | `--bwlimit=KB/s` | 0 (unlimited) |
| Filter rules | `--include`/`--exclude` (ordered) | none |
| Extra flags | whitelisted subset | none |

## Filter Rules

The options modal has a VS Code-style filter rules editor:
- Each rule is **include** or **exclude** with a pattern
- Rules are ordered — first match wins (drag to reorder)
- Preset buttons: git, node_modules, pycache, .DS_Store, *.pyc, .env, .vscode, logs, .idea, objects, dist/build, temp
- Profiles save/restore filter rules

### Pattern Syntax (rsync glob)

| Pattern | Meaning |
|---------|---------|
| `*` | Any chars except `/` |
| `**` | Any chars including `/` |
| `?` | Single char |
| `[abc]` | Character class |
| Leading `/` | Anchors to transfer root |
| Trailing `/` | Matches directories only |

## .rsyncignore

When "read .rsyncignore" is enabled (default), rsync reads a `.rsyncignore` file in each directory it traverses. Rules apply to that directory and its subdirectories.

### Format

Plain list of exclude patterns, one per line. Same glob syntax as filter rules above.

```
# Example .rsyncignore
__pycache__/
*.pyc
.env
node_modules/
*.log
.git/
dist/
build/
*.tmp
*.swp
```

### Behavior

- **Per-directory**: each folder can have its own `.rsyncignore`
- **Inherited**: rules apply downward into subdirectories
- **Wildcards**: `*`, `**`, `?`, `[...]` all work
- **Directory-only**: trailing `/` matches only directories
- **Anchoring**: leading `/` anchors to that directory only

### Differences from .gitignore

| Feature | .gitignore | .rsyncignore |
|---------|-----------|--------------|
| Negation | `!pattern` | `+ pattern` (include prefix) |
| Exclude prefix | implicit | `- pattern` (optional, default) |
| Include prefix | `!pattern` | `+ pattern` |
| Auto-inherit | parent dirs | parent dirs (via dir-merge) |

For simple exclude-only use, syntax is identical to `.gitignore`.

### Advanced: mixed include/exclude

```
# .rsyncignore with include/exclude
+ *.py
+ *.md
- *
```

This syncs only `.py` and `.md` files, excluding everything else.

## Remote Path Handling

- Remote paths starting with `~/` are converted to relative paths (rsync resolves from home dir)
- `~` alone becomes `.`
- No shell quoting needed — command built as Python list

## Exit Code Handling

| Code | Status | Meaning |
|------|--------|---------|
| 0 | done | Success |
| 23 | done (warning) | Partial transfer — some files skipped |
| 24 | done (warning) | Some files vanished during transfer |
| 20 | cancelled | SIGTERM received |
| other | error | Real failure |

## Security

- Local paths: must be absolute, no `..` traversal
- SSH keys: must exist, no path traversal
- Extra flags: whitelisted subset only (`--checksum`, `--ignore-existing`, `--update`, etc.)
- Remote paths: no shell injection (subprocess list, not string)