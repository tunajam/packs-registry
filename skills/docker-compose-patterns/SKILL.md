# Docker Compose Patterns for Homelab

Production-ready patterns for self-hosted services using Docker Compose.

## When to Use

- Setting up new homelab services
- Organizing multi-container applications
- Configuring networks, volumes, and dependencies
- Troubleshooting container issues

## Core Patterns

### 1. Basic Service Structure

```yaml
services:
  myapp:
    image: ghcr.io/org/app:latest
    container_name: myapp
    restart: unless-stopped
    environment:
      - TZ=America/Los_Angeles
      - PUID=1000
      - PGID=1000
    volumes:
      - ./config:/config
      - ./data:/data
    ports:
      - "8080:80"
```

### 2. LinuxServer.io Pattern (PUID/PGID)

Most homelab containers follow this pattern:

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000        # Your user ID (run `id -u`)
      - PGID=1000        # Your group ID (run `id -g`)
      - TZ=America/Los_Angeles
    volumes:
      - /path/to/config:/config
      - /path/to/media:/media
    ports:
      - 8096:8096
    restart: unless-stopped
```

### 3. Custom Networks (Isolated Services)

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No internet access

services:
  nginx:
    networks:
      - frontend
      - backend
  
  app:
    networks:
      - backend
  
  db:
    networks:
      - backend
```

### 4. Named Volumes vs Bind Mounts

```yaml
volumes:
  db_data:      # Named volume - Docker manages location
  cache_data:

services:
  postgres:
    volumes:
      - db_data:/var/lib/postgresql/data     # Named volume (persistent)
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # Bind mount (read-only)
```

**Rule of thumb:**
- **Named volumes**: Database data, persistent application state
- **Bind mounts**: Config files, media, things you need to edit

### 5. Healthchecks & Dependencies

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    depends_on:
      db:
        condition: service_healthy
```

### 6. Environment Files

```yaml
services:
  app:
    env_file:
      - .env           # Default
      - secrets.env    # Additional secrets
    environment:
      - PUBLIC_VAR=value  # Overrides .env
```

`.env` file:
```env
POSTGRES_USER=admin
POSTGRES_PASSWORD=changeme
POSTGRES_DB=myapp
```

### 7. Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
```

### 8. Traefik Labels Pattern

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
```

### 9. Logging Configuration

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 10. GPU Passthrough (Jellyfin/Plex Transcoding)

```yaml
services:
  jellyfin:
    devices:
      - /dev/dri:/dev/dri  # Intel QuickSync
    # OR for NVIDIA:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

## Complete Stack Example

```yaml
# Media server stack with Traefik
networks:
  proxy:
    external: true
  media:
    driver: bridge

volumes:
  jellyfin_config:

services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - jellyfin_config:/config
      - /mnt/media/movies:/data/movies
      - /mnt/media/tv:/data/tv
    devices:
      - /dev/dri:/dev/dri
    networks:
      - proxy
      - media
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`media.example.com`)"
      - "traefik.http.routers.jellyfin.entrypoints=websecure"
      - "traefik.http.routers.jellyfin.tls.certresolver=letsencrypt"
      - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
    restart: unless-stopped
    healthcheck:
      test: curl -f http://localhost:8096/health || exit 1
      interval: 30s
      timeout: 10s
      retries: 3
```

## Common Commands

```bash
# Start services (detached)
docker compose up -d

# View logs
docker compose logs -f [service]

# Restart single service
docker compose restart jellyfin

# Pull latest images and recreate
docker compose pull && docker compose up -d

# Remove everything including volumes
docker compose down -v

# Execute command in container
docker compose exec jellyfin bash

# View resource usage
docker stats
```

## Best Practices

1. **Always use `restart: unless-stopped`** - survives reboots but respects manual stops
2. **Pin image versions for production** - `image: postgres:16.2` not `postgres:latest`
3. **Use `.env` for secrets** - never commit to git, add to `.gitignore`
4. **Set resource limits** - prevent runaway containers from killing your server
5. **Use healthchecks** - helps with dependency ordering and monitoring
6. **Isolate databases** - internal networks, no direct port exposure
7. **Log rotation** - prevent disk fill from chatty containers
8. **Use PUID/PGID** - match your host user to avoid permission issues
