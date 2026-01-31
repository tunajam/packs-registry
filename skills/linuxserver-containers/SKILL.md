# LinuxServer.io Containers

The most popular container images for homelab, maintained by LinuxServer.io.

## When to Use

- Deploying media servers (Plex, Jellyfin, Sonarr, Radarr)
- Self-hosted apps (Nextcloud, Gitea, Bookstack)
- Need consistent, well-maintained container images

## Core Concepts

### PUID and PGID

All LinuxServer containers use PUID/PGID to run as your user:

```bash
# Find your user/group IDs
id $USER
# uid=1000(fred) gid=1000(fred) groups=...
```

```yaml
environment:
  - PUID=1000
  - PGID=1000
  - TZ=America/Los_Angeles
```

**Why?** Files created inside the container will be owned by your user, not root.

### Standard Paths

All LinuxServer images use consistent paths:

| Path | Purpose |
|------|---------|
| `/config` | Application config and data |
| `/data` | General data (varies by app) |
| `/downloads` | Download location |
| `/media` | Media files |

### Image Registry

```yaml
# Use lscr.io (LinuxServer Container Registry)
image: lscr.io/linuxserver/jellyfin:latest

# Also available on Docker Hub
image: linuxserver/jellyfin:latest

# Pin specific version
image: lscr.io/linuxserver/jellyfin:10.8.13
```

## Docker Mods

Extend container functionality without rebuilding:

```yaml
services:
  nginx:
    image: lscr.io/linuxserver/nginx
    environment:
      - DOCKER_MODS=lscr.io/linuxserver/mods:universal-cloudflared
      - CF_ZONE_ID=your_zone_id
      - CF_ACCOUNT_ID=your_account_id
      - CF_API_TOKEN=your_token
```

### Popular Mods

```yaml
# Cloudflare Tunnel
DOCKER_MODS=lscr.io/linuxserver/mods:universal-cloudflared

# Calibre with Kepubify
DOCKER_MODS=lscr.io/linuxserver/mods:calibre-kepubify

# Multiple mods (pipe-separated)
DOCKER_MODS=lscr.io/linuxserver/mods:mod1|lscr.io/linuxserver/mods:mod2
```

Browse available mods: https://mods.linuxserver.io/

## Custom Scripts

Run scripts at container startup:

```yaml
volumes:
  - ./custom-init:/custom-cont-init.d:ro
```

`custom-init/install-ffmpeg.sh`:
```bash
#!/bin/bash
echo "**** installing ffmpeg ****"
apk add --no-cache ffmpeg
```

**Important:** Scripts must be executable (`chmod +x`)

## Custom Services

Run additional services inside the container:

```yaml
volumes:
  - ./custom-services:/custom-services.d:ro
```

`custom-services/memcached`:
```bash
#!/usr/bin/with-contenv bash
exec memcached -u abc
```

## Complete Examples

### Jellyfin Media Server

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
      - /mnt/media/movies:/data/movies
      - /mnt/media/tv:/data/tv
    ports:
      - 8096:8096
      - 8920:8920  # HTTPS
      - 7359:7359/udp  # Discovery
    devices:
      - /dev/dri:/dev/dri  # Intel QuickSync
    restart: unless-stopped
```

### Sonarr + Radarr + Prowlarr Stack

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./sonarr:/config
      - /mnt/media/tv:/tv
      - /mnt/downloads:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./radarr:/config
      - /mnt/media/movies:/movies
      - /mnt/downloads:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

### Nextcloud

```yaml
services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
      - ./data:/data
    ports:
      - 443:443
    restart: unless-stopped
```

## Troubleshooting

### Permission Denied Errors

```bash
# Check ownership
ls -la /path/to/config

# Fix ownership (match PUID/PGID)
sudo chown -R 1000:1000 /path/to/config
```

### Finding Container Logs

```bash
# Docker logs
docker logs -f jellyfin

# Application logs inside container
docker exec -it jellyfin cat /config/logs/log.txt
```

### Container Won't Start

```bash
# Check events
docker events --filter container=jellyfin

# Inspect container
docker inspect jellyfin | grep -A 10 State
```

## Best Practices

1. **Always set PUID/PGID** to your actual user IDs
2. **Use bind mounts for media** - easier to manage large libraries
3. **Use named volumes for config** - cleaner backups
4. **Set timezone (TZ)** - prevents log timestamp confusion
5. **Pin versions in production** - `jellyfin:10.8.13` not `:latest`
6. **Check image docs** - each app has specific environment variables
