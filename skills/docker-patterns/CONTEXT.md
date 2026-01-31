# Docker Patterns

Best practices for Dockerfiles, multi-stage builds, security, and deployment.

## When to Apply

- Writing efficient Dockerfiles
- Optimizing image size and build time
- Implementing security best practices
- Creating Docker Compose configurations
- Debugging container issues
- Setting up CI/CD pipelines

## Multi-Stage Builds

### Go Application
```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /app/server ./cmd/server

# Runtime stage
FROM alpine:3.19

RUN apk --no-cache add ca-certificates

WORKDIR /app

# Copy binary only
COPY --from=builder /app/server .

# Non-root user
RUN adduser -D -u 1000 appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["./server"]
```

### Node.js Application
```dockerfile
# Dependencies stage
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage
FROM node:20-alpine
WORKDIR /app

ENV NODE_ENV=production

# Copy production deps
COPY --from=deps /app/node_modules ./node_modules
# Copy built app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001
USER nextjs

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Python Application
```dockerfile
# Build stage
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN pip install --no-cache-dir poetry

COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes

# Runtime stage
FROM python:3.12-slim

WORKDIR /app

# Install runtime dependencies only
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Non-root user
RUN useradd -m -u 1000 appuser
USER appuser

EXPOSE 8000

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0"]
```

## Layer Optimization

### Order Matters (Cache)
```dockerfile
# Bad: COPY all files early invalidates cache
COPY . .
RUN npm install

# Good: Copy dependency files first
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

### Combine RUN Commands
```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# Good: Single layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

### .dockerignore
```dockerignore
# Dependencies
node_modules/
vendor/
.venv/

# Build artifacts
dist/
build/
*.pyc
__pycache__/

# Git
.git/
.gitignore

# IDE
.vscode/
.idea/

# Local config
.env
.env.local
*.local

# Tests
coverage/
.pytest_cache/

# Docker
Dockerfile*
docker-compose*.yml
.docker/
```

## Security Best Practices

### Non-Root User
```dockerfile
# Create and switch to non-root user
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --gid 1001 appuser

# Set ownership
COPY --chown=appuser:appgroup . .

USER appuser
```

### Read-Only Filesystem
```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

### No Secrets in Images
```dockerfile
# Bad: Secret in image
ENV API_KEY=secret123

# Good: Runtime secrets
# Pass via environment variable at runtime
# docker run -e API_KEY=secret123 myapp

# Better: Docker secrets (Swarm) or external secret manager
```

### Minimal Base Images
```dockerfile
# Size comparison (approximate):
# ubuntu:22.04     ~77MB
# debian:slim      ~80MB
# alpine:3.19      ~7MB
# gcr.io/distroless/static-debian12  ~2MB
# scratch          ~0MB

# For Go (static binary)
FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# For most apps
FROM alpine:3.19

# For distroless (no shell)
FROM gcr.io/distroless/base-debian12
```

### Scan for Vulnerabilities
```bash
# Docker Scout
docker scout cves myimage:latest

# Trivy
trivy image myimage:latest

# Snyk
snyk container test myimage:latest
```

## Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Or for apps without curl
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
```

## Docker Compose

### Development Setup
```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules  # Don't overwrite node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://postgres:password@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Production Setup
```yaml
# docker-compose.prod.yml
services:
  app:
    image: myapp:${VERSION:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - NODE_ENV=production
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
```

### Override Files
```bash
# Base + development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Base + production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Networking

### Custom Networks
```yaml
services:
  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access
```

### Service Discovery
```yaml
# Services can reach each other by service name
# app can connect to db:5432
services:
  app:
    environment:
      - DATABASE_HOST=db
  db:
    image: postgres:16
```

## Volume Patterns

### Named Volumes
```yaml
volumes:
  postgres_data:
    driver: local
  uploads:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/uploads
```

### Bind Mounts (Development)
```yaml
services:
  app:
    volumes:
      - ./src:/app/src:ro    # Read-only source
      - ./config:/app/config  # Read-write config
```

### tmpfs (Sensitive Data)
```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /run/secrets  # Secrets disappear on stop
```

## Build Arguments & Env Vars

```dockerfile
# Build-time args
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG BUILD_DATE
ARG GIT_SHA
LABEL build_date=$BUILD_DATE git_sha=$GIT_SHA

# Runtime env (can be overridden)
ENV PORT=3000
ENV NODE_ENV=production
```

```bash
# Build with args
docker build \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --build-arg GIT_SHA=$(git rev-parse --short HEAD) \
  -t myapp:latest .
```

## Debugging

### Interactive Shell
```bash
# Running container
docker exec -it container_name /bin/sh

# New container from image
docker run -it --rm myimage /bin/sh

# Override entrypoint
docker run -it --rm --entrypoint /bin/sh myimage
```

### View Logs
```bash
docker logs -f container_name
docker logs --tail 100 container_name
docker logs --since 1h container_name
```

### Inspect
```bash
# Container details
docker inspect container_name

# Image layers
docker history myimage

# Filesystem changes
docker diff container_name
```

### Resource Usage
```bash
docker stats container_name
docker top container_name
```

## CI/CD Patterns

### GitHub Actions
```yaml
name: Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### BuildKit Cache
```dockerfile
# syntax=docker/dockerfile:1.5

FROM node:20-alpine

WORKDIR /app

# Cache npm packages
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .
RUN npm run build
```

## Common Issues

### Container Exits Immediately
```dockerfile
# Problem: Process runs in background
CMD ["service", "start"]

# Solution: Run in foreground
CMD ["service", "-foreground"]

# Or keep alive for debugging
CMD ["tail", "-f", "/dev/null"]
```

### Permission Denied
```bash
# Check file ownership in image
docker run --rm myimage ls -la /app

# Fix in Dockerfile
RUN chown -R appuser:appuser /app
USER appuser
```

### DNS Issues
```yaml
# docker-compose.yml
services:
  app:
    dns:
      - 8.8.8.8
      - 8.8.4.4
```

## References

- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Security](https://docs.docker.com/engine/security/)
