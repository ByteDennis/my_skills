---
name: docker-workflow
description: Docker and docker compose common workflows
tags: [docker, containers, deployment, local]
---

# Docker Workflow

## Quick Aliases

| Alias | Command |
|-------|---------|
| `dcc` | `docker compose` |
| `dps` | `docker ps` (formatted table) |
| `dlog` | `docker logs -f` |
| `lzd` | `lazydocker` (TUI) |

## Common Workflows

### Build and restart a service
```bash
docker compose build <service> && docker compose up -d <service>
```

### View logs with follow
```bash
docker logs -f --tail 100 <container>
```

### Shell into a running container
```bash
docker exec -it <container> bash
```

### Clean up unused resources
```bash
docker system prune -af --volumes
```

### Copy files from container
```bash
docker cp <container>:/path/to/file ./local/path
```

### Check resource usage
```bash
docker stats --no-stream
```

## Running Services (port 5001)

The clipboard-upload Flask app runs as `files-clipboard-upload` container:
- Clipboard upload, Scholar AI, Podcast RSS, Draw SVG, Overleaf AI, Rsync, Skill Cards
- Data persisted at `/data/` inside container
- Rebuild: `docker build -t files-clipboard-upload . && docker compose up -d`