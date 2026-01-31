# Traefik for Homelab

Traefik is a dynamic reverse proxy that auto-discovers Docker containers.

## When to Use

- Docker-based homelab with many services
- Need automatic service discovery via labels
- Want middleware (auth, rate limiting, headers)
- Multi-host deployments

## Why Traefik?

- **Auto-discovery** - containers expose themselves via labels
- **Dynamic config** - no reload needed when adding services
- **Built-in dashboard** - visual overview of routes
- **Middleware** - authentication, headers, rate limiting built-in

## Docker Compose Setup

### Traefik Container

```yaml
networks:
  proxy:
    external: true

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    environment:
      - CF_DNS_API_TOKEN=${CF_API_TOKEN}  # For DNS challenge
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./acme.json:/acme.json
      - ./config:/config
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.example.com`)"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."
```

### traefik.yml

```yaml
api:
  dashboard: true
  insecure: false

log:
  level: INFO

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: proxy
  file:
    directory: /config
    watch: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

## Service Labels

### Basic Service

```yaml
services:
  whoami:
    image: traefik/whoami
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
```

### Custom Port

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.example.com`)"
  - "traefik.http.routers.jellyfin.entrypoints=websecure"
  - "traefik.http.routers.jellyfin.tls.certresolver=letsencrypt"
  - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
```

### Path-Based Routing

```yaml
labels:
  - "traefik.http.routers.api.rule=Host(`example.com`) && PathPrefix(`/api`)"
```

### Multiple Domains

```yaml
labels:
  - "traefik.http.routers.app.rule=Host(`app.example.com`) || Host(`app.example.org`)"
```

## Middleware

### Basic Auth

```yaml
labels:
  - "traefik.http.middlewares.myauth.basicauth.users=admin:$$apr1$$xyz..."
  - "traefik.http.routers.app.middlewares=myauth"
```

Generate password:
```bash
htpasswd -nB admin | sed -e 's/\$/\$\$/g'
```

### Headers

```yaml
labels:
  - "traefik.http.middlewares.secheaders.headers.stsSeconds=31536000"
  - "traefik.http.middlewares.secheaders.headers.stsIncludeSubdomains=true"
  - "traefik.http.middlewares.secheaders.headers.frameDeny=true"
  - "traefik.http.routers.app.middlewares=secheaders"
```

### Rate Limiting

```yaml
labels:
  - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
  - "traefik.http.middlewares.ratelimit.ratelimit.burst=50"
  - "traefik.http.routers.app.middlewares=ratelimit"
```

### IP Whitelist

```yaml
labels:
  - "traefik.http.middlewares.local.ipwhitelist.sourcerange=192.168.0.0/16,10.0.0.0/8"
  - "traefik.http.routers.app.middlewares=local"
```

### Chain Multiple Middleware

```yaml
labels:
  - "traefik.http.routers.app.middlewares=auth,secheaders,ratelimit"
```

## Authelia Integration

### Dynamic Config (config/authelia.yml)

```yaml
http:
  middlewares:
    authelia:
      forwardAuth:
        address: "http://authelia:9091/api/verify?rd=https://auth.example.com"
        trustForwardHeader: true
        authResponseHeaders:
          - "Remote-User"
          - "Remote-Groups"
          - "Remote-Name"
          - "Remote-Email"
```

### Apply to Service

```yaml
labels:
  - "traefik.http.routers.app.middlewares=authelia@file"
```

## Wildcard Certificates

```yaml
# In traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme.json
      dnsChallenge:
        provider: cloudflare
```

```yaml
# Service labels
labels:
  - "traefik.http.routers.app.tls.certresolver=letsencrypt"
  - "traefik.http.routers.app.tls.domains[0].main=example.com"
  - "traefik.http.routers.app.tls.domains[0].sans=*.example.com"
```

## External Services (Non-Docker)

### config/external.yml

```yaml
http:
  routers:
    proxmox:
      rule: "Host(`proxmox.example.com`)"
      service: proxmox
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
  
  services:
    proxmox:
      loadBalancer:
        servers:
          - url: "https://192.168.1.10:8006"
        passHostHeader: true
```

## Troubleshooting

### Debug Mode

```yaml
# traefik.yml
log:
  level: DEBUG

accessLog: {}
```

### Common Issues

**404 Not Found:**
- Check `traefik.enable=true` label
- Verify container is on the `proxy` network
- Check router rule syntax

**502 Bad Gateway:**
- Service port not specified (add `loadbalancer.server.port`)
- Container not healthy
- Network connectivity

**Certificate Errors:**
- Check acme.json permissions (`chmod 600 acme.json`)
- Verify DNS challenge token
- Check rate limits on Let's Encrypt

### View Current Config

```bash
# Access API
curl http://localhost:8080/api/rawdata

# Or use dashboard at traefik.example.com
```

## Best Practices

1. **Create external network** - `docker network create proxy`
2. **Never expose Docker socket directly** - use socket proxy if paranoid
3. **Use DNS challenge** - works behind firewall, supports wildcards
4. **Set `exposedByDefault: false`** - explicit opt-in per service
5. **Backup acme.json** - contains certificates
6. **Use file provider for external services** - non-Docker hosts
7. **Pin Traefik version** - `traefik:v3.0` not `traefik:latest`
