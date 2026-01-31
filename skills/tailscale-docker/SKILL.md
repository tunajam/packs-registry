# tailscale-docker

Tailscale Docker integration - sidecar patterns, container networking, Compose

## Source

Tailscale Docker documentation: https://tailscale.com/kb/1282/docker

## Docker Image

```bash
# From Docker Hub
docker pull tailscale/tailscale:stable

# From GitHub Packages
docker pull ghcr.io/tailscale/tailscale:stable
```

### Tags
- `stable` / `latest` - Latest stable version
- `v1.58.2`, `v1.58` - Specific versions
- `unstable` - Latest unstable version

## Environment Variables

### Authentication
| Variable | Description |
|----------|-------------|
| `TS_AUTHKEY` | Auth key for login |
| `TS_HOSTNAME` | Device hostname |
| `TS_AUTH_ONCE` | Only auth if not already logged in |

### Networking
| Variable | Description |
|----------|-------------|
| `TS_USERSPACE` | Enable userspace networking (default: true) |
| `TS_ACCEPT_DNS` | Accept DNS from Tailscale |
| `TS_ROUTES` | Advertise subnet routes |
| `TS_DEST_IP` | Proxy traffic to this IP |
| `TS_EXTRA_ARGS` | Additional tailscale up args |

### State & Proxy
| Variable | Description |
|----------|-------------|
| `TS_STATE_DIR` | State directory path |
| `TS_KUBE_SECRET` | Kubernetes secret for state |
| `TS_SOCKS5_SERVER` | SOCKS5 proxy address |
| `TS_OUTBOUND_HTTP_PROXY_LISTEN` | HTTP proxy address |

### Serve & Funnel
| Variable | Description |
|----------|-------------|
| `TS_SERVE_CONFIG` | JSON config for Serve/Funnel |

### Health & Metrics
| Variable | Description |
|----------|-------------|
| `TS_ENABLE_HEALTH_CHECK` | Enable /healthz endpoint |
| `TS_ENABLE_METRICS` | Enable /metrics endpoint |
| `TS_LOCAL_ADDR_PORT` | Health/metrics listen address (default: [::]:9002) |

## Basic Docker Run

```bash
docker run -d \
  --name tailscale \
  --hostname my-container \
  -v /dev/net/tun:/dev/net/tun \
  -v tailscale-state:/var/lib/tailscale \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  -e TS_AUTHKEY=tskey-auth-xxxxx \
  -e TS_HOSTNAME=my-container \
  tailscale/tailscale:stable
```

## Docker Compose Examples

### Basic Tailscale Container
```yaml
version: "3.9"
services:
  tailscale:
    image: tailscale/tailscale:stable
    hostname: my-service
    environment:
      - TS_AUTHKEY=tskey-auth-xxxxx
      - TS_HOSTNAME=my-service
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - tailscale-state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    restart: unless-stopped

volumes:
  tailscale-state:
```

### Sidecar Pattern
Expose another container through Tailscale:

```yaml
version: "3.9"
services:
  tailscale:
    image: tailscale/tailscale:stable
    hostname: my-nginx
    environment:
      - TS_AUTHKEY=tskey-auth-xxxxx
      - TS_HOSTNAME=my-nginx
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
    volumes:
      - tailscale-state:/var/lib/tailscale
      - ./serve.json:/config/serve.json:ro
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    network_mode: service:nginx  # Share network with nginx

  nginx:
    image: nginx:alpine
    # No ports exposed - accessed via Tailscale

volumes:
  tailscale-state:
```

### Subnet Router
```yaml
version: "3.9"
services:
  tailscale-router:
    image: tailscale/tailscale:stable
    hostname: docker-router
    environment:
      - TS_AUTHKEY=tskey-auth-xxxxx
      - TS_HOSTNAME=docker-router
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_ROUTES=172.20.0.0/16
      - TS_EXTRA_ARGS=--advertise-tags=tag:router
    volumes:
      - tailscale-state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      - app-network

  app:
    image: my-app
    networks:
      - app-network

networks:
  app-network:
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  tailscale-state:
```

### Exit Node
```yaml
version: "3.9"
services:
  tailscale-exit:
    image: tailscale/tailscale:stable
    hostname: docker-exit
    environment:
      - TS_AUTHKEY=tskey-auth-xxxxx
      - TS_HOSTNAME=docker-exit
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_EXTRA_ARGS=--advertise-exit-node --advertise-tags=tag:exit
    volumes:
      - tailscale-state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv6.conf.all.forwarding=1

volumes:
  tailscale-state:
```

### With OAuth Client
```yaml
version: "3.9"
services:
  tailscale:
    image: tailscale/tailscale:stable
    hostname: my-service
    environment:
      - TS_HOSTNAME=my-service
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_EXTRA_ARGS=--advertise-tags=tag:container
    env_file:
      - .env  # Contains OAuth credentials
    volumes:
      - tailscale-state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW

volumes:
  tailscale-state:
```

`.env` file:
```
TS_AUTHKEY=tskey-client-xxxxx-secret
```

## Serve Config JSON

For `TS_SERVE_CONFIG`:

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "my-service.tail12345.ts.net:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:3000"
        }
      }
    }
  }
}
```

## Userspace vs Kernel Mode

### Userspace (Default)
- No `/dev/net/tun` needed
- Fewer privileges required
- Slightly lower performance
- Works in restricted environments

```yaml
environment:
  - TS_USERSPACE=true
# No tun device or NET_ADMIN needed
```

### Kernel Mode
- Better performance
- Requires `/dev/net/tun` and NET_ADMIN
- Full networking features

```yaml
volumes:
  - /dev/net/tun:/dev/net/tun
cap_add:
  - NET_ADMIN
  - NET_RAW
environment:
  - TS_USERSPACE=false
```

## Health Checks

```yaml
services:
  tailscale:
    image: tailscale/tailscale:stable
    environment:
      - TS_ENABLE_HEALTH_CHECK=true
      - TS_LOCAL_ADDR_PORT=:9002
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9002/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## Ephemeral Containers

For CI/CD or temporary workloads:

```yaml
environment:
  - TS_AUTHKEY=tskey-auth-xxxxx?ephemeral=true
```

Container auto-removed from tailnet when stopped.

## SOCKS5/HTTP Proxy

Access tailnet from non-Tailscale containers:

```yaml
services:
  tailscale:
    image: tailscale/tailscale:stable
    environment:
      - TS_SOCKS5_SERVER=:1055
      - TS_OUTBOUND_HTTP_PROXY_LISTEN=:1080
    ports:
      - "1055:1055"
      - "1080:1080"

  app:
    image: my-app
    environment:
      - HTTP_PROXY=http://tailscale:1080
      - HTTPS_PROXY=http://tailscale:1080
```

## Best Practices

1. **Use auth keys** - Automate container authentication
2. **Persist state** - Mount volume for `/var/lib/tailscale`
3. **Use ephemeral for CI** - Auto-cleanup temporary containers
4. **Tag containers** - Use `--advertise-tags` for ACL management
5. **Enable health checks** - Monitor container connectivity
6. **Use OAuth for long-running** - More secure than static auth keys

## Troubleshooting

### Container won't authenticate
```bash
# Check logs
docker logs tailscale

# Verify auth key is valid
# Keys expire - generate new if needed
```

### Can't reach container
```bash
# Inside container
docker exec tailscale tailscale status
docker exec tailscale tailscale ping <target>
```

### State not persisting
- Ensure volume is mounted to `/var/lib/tailscale`
- Check volume permissions
