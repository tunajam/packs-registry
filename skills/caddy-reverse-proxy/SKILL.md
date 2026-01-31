# Caddy Reverse Proxy for Homelab

Caddy is the easiest reverse proxy with automatic HTTPS.

## When to Use

- Need automatic SSL certificates (Let's Encrypt)
- Simple configuration requirements
- Single server homelab setup
- Want HTTPS without complexity

## Why Caddy?

- **Automatic HTTPS** - certificates provisioned and renewed automatically
- **Simple config** - Caddyfile is human-readable
- **HTTP/2 and HTTP/3** - enabled by default
- **Zero downtime reloads** - config changes apply instantly

## Basic Patterns

### Simple Reverse Proxy

```caddyfile
jellyfin.example.com {
    reverse_proxy localhost:8096
}

nextcloud.example.com {
    reverse_proxy localhost:443 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

### Static Files + API Proxy

```caddyfile
example.com {
    root * /var/www/html
    reverse_proxy /api/* localhost:3000
    file_server
}
```

### SPA (Single Page App)

```caddyfile
app.example.com {
    root * /srv/app
    encode gzip
    try_files {path} /index.html
    file_server
}
```

### Wildcard Subdomain Proxy

```caddyfile
*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }

    @jellyfin host jellyfin.example.com
    handle @jellyfin {
        reverse_proxy localhost:8096
    }

    @sonarr host sonarr.example.com
    handle @sonarr {
        reverse_proxy localhost:8989
    }

    handle {
        abort
    }
}
```

### Basic Authentication

```caddyfile
private.example.com {
    basicauth {
        admin $2a$14$hashedpassword...
    }
    reverse_proxy localhost:8080
}
```

Generate password hash:
```bash
caddy hash-password
```

### Headers for Security

```caddyfile
(security_headers) {
    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        -Server
    }
}

app.example.com {
    import security_headers
    reverse_proxy localhost:3000
}
```

## Docker Compose Setup

```yaml
services:
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"  # HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./site:/srv
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - CF_API_TOKEN=${CF_API_TOKEN}  # For DNS challenge

volumes:
  caddy_data:
  caddy_config:
```

## DNS Challenge (Wildcard Certs)

For internal/Tailscale domains or wildcards, use DNS challenge:

```caddyfile
{
    acme_dns cloudflare {env.CF_API_TOKEN}
}

*.home.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
    
    @jellyfin host jellyfin.home.example.com
    handle @jellyfin {
        reverse_proxy 192.168.1.50:8096
    }
}
```

### Custom Caddy with DNS Module

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

## Internal Services (Tailscale/Local)

```caddyfile
{
    # Disable automatic HTTPS for local domains
    local_certs
}

jellyfin.local {
    tls internal
    reverse_proxy 192.168.1.50:8096
}
```

## Snippets (Reusable Config)

```caddyfile
(proxy_headers) {
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-For {remote_host}
    header_up X-Forwarded-Proto {scheme}
}

(cloudflare) {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
}

jellyfin.example.com {
    import cloudflare
    reverse_proxy localhost:8096 {
        import proxy_headers
    }
}
```

## Authelia Integration

```caddyfile
(authelia) {
    forward_auth localhost:9091 {
        uri /api/verify?rd=https://auth.example.com
        copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
    }
}

private.example.com {
    import authelia
    reverse_proxy localhost:8080
}
```

## Common Commands

```bash
# Validate Caddyfile
caddy validate --config /etc/caddy/Caddyfile

# Reload config (zero downtime)
caddy reload --config /etc/caddy/Caddyfile

# Format Caddyfile
caddy fmt --overwrite /etc/caddy/Caddyfile

# Run in foreground
caddy run --config /etc/caddy/Caddyfile

# View certificate info
caddy trust
```

## Troubleshooting

### Certificate Issues

```bash
# Check certificate status
curl -vI https://yoursite.example.com 2>&1 | grep -A 5 "Server certificate"

# Force certificate renewal
docker exec caddy caddy reload --config /etc/caddy/Caddyfile --force
```

### 502 Bad Gateway

1. Check if upstream service is running
2. Verify port mapping
3. Check container networking (use container name, not localhost)

```caddyfile
# For Docker services, use container name
jellyfin.example.com {
    reverse_proxy jellyfin:8096  # Not localhost:8096
}
```

### Access Logs

```caddyfile
{
    log {
        output file /var/log/caddy/access.log
        format json
    }
}
```

## Best Practices

1. **Use DNS challenge for wildcards** - HTTP challenge doesn't support wildcards
2. **Pin Caddy version** - avoid surprise breaking changes
3. **Use snippets** - DRY principle for repeated config
4. **Always use HTTPS** - Caddy makes this trivial
5. **Rate limiting for public services** - add `rate_limit` directive
6. **Backup /data volume** - contains certificates
