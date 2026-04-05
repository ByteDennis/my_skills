---
name: server-admin
description: Server administration commands and system monitoring
tags: [server, admin, systemd, networking, local]
---

# Server Administration

## System Monitoring

| Alias | Command |
|-------|---------|
| `top` | `btop` (better top) |
| `df` | `duf` (better df) |
| `mem` | `free -h` |
| `ports` | `ss -tulnp` |
| `gpu` | `nvidia-smi` |

## Systemd

```bash
# Using sc alias (sudo systemctl with nvim editor)
sc status <service>
sc restart <service>
sc enable <service>
sc journal -u <service> -f    # follow logs
```

## Networking

```bash
# Check what's listening on a port
ports | grep :5001

# Kill process on a port
killport 5001

# Quick connectivity test
curl -s -o /dev/null -w "%{http_code}" http://localhost:5001/
```

## Firewall (UFW)

```bash
sudo ufw status
sudo ufw allow 5001/tcp
sudo ufw delete allow 5001/tcp
```

## Disk Usage

```bash
dsize /data           # top 20 dirs by size (uses dust)
findlarge 1G          # find files > 1GB
duf                   # colorful disk usage overview
```

## SSH to Other Servers

```bash
ssh xie               # uses ~/.ssh/config aliases
rsync -avz --progress src/ xie:~/dest/
```