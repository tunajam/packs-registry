# Nginx Proxy Manager

Easy reverse proxy with web UI and automatic SSL.

## When to Use

- Need reverse proxy without CLI
- Want visual SSL certificate management
- Simple homelab with few services
- Prefer GUI over config files

## Docker Setup

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"     # HTTP
      - "443:443"   # HTTPS
      - "81:81"     # Admin UI
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      - TZ=America/Los_Angeles
```

### First Run

1. Navigate to `http://localhost:81`
2. Default credentials:
   - Email: `admin@example.com`
   - Password: `changeme`
3. Change password immediately

## Proxy Hosts

### Add Proxy Host

Hosts → Proxy Hosts → Add:

**Details:**
- Domain: `jellyfin.example.com`
- Scheme: `http`
- Forward Hostname: `192.168.1.50`
- Forward Port: `8096`
- Cache Assets: Yes
- Websockets Support: Yes (for real-time apps)

**SSL:**
- Request new SSL certificate
- Force SSL
- HTTP/2 Support

### Common Services

| Service | Port | Websockets |
|---------|------|------------|
| Jellyfin | 8096 | Yes |
| Nextcloud | 443 | No |
| Home Assistant | 8123 | Yes |
| Grafana | 3000 | Yes |
| Portainer | 9000 | Yes |

## SSL Certificates

### Let's Encrypt

SSL Certificates → Add:
- Domain: `*.example.com`
- Email: your email
- Use DNS Challenge: Yes (for wildcard)
- DNS Provider: Cloudflare

### DNS Challenge Providers

**Cloudflare:**
```
CF_Token = your-api-token
```

**Route53:**
```
AWS_ACCESS_KEY_ID = your-key
AWS_SECRET_ACCESS_KEY = your-secret
```

### Use Existing Certificate

SSL Certificates → Add → Custom:
- Upload certificate and key files
- Useful for corporate certs

## Access Lists

### Create Access List

Access Lists → Add:
- Name: "Internal Only"
- Satisfy Any: No

**Authorization:**
- Add users for basic auth

**Access:**
- Allow: `192.168.1.0/24`
- Deny: all

### Apply to Host

Hosts → [Host] → Edit → Access List: "Internal Only"

## Custom Locations

### API Forwarding

Hosts → [Host] → Custom Locations:
- Location: `/api`
- Scheme: `http`
- Forward Hostname: `backend`
- Forward Port: `3000`

### Static Files

Location: `/static`
- Forward to different server
- Or serve from local path

## Advanced Configuration

### Custom Nginx Config

Hosts → [Host] → Advanced:
```nginx
# Increase upload size
client_max_body_size 100M;

# Custom headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;

# Proxy headers
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### Disable Caching

```nginx
proxy_no_cache 1;
proxy_cache_bypass 1;
```

### Timeout Adjustments

```nginx
proxy_connect_timeout 300;
proxy_send_timeout 300;
proxy_read_timeout 300;
```

## Redirection Hosts

### WWW Redirect

Hosts → Redirection Hosts → Add:
- Domain: `www.example.com`
- Scheme: `https`
- Domain: `example.com`
- Preserve Path: Yes

### HTTP to HTTPS

Automatic when "Force SSL" is enabled on proxy host.

## 404 Hosts

Catch-all for undefined subdomains:

Hosts → 404 Hosts → Add:
- Domain: `*.example.com`

## Streams

For TCP/UDP proxying (not HTTP):

Hosts → Streams → Add:
- Incoming Port: `25565` (Minecraft)
- Forward Host: `192.168.1.60`
- Forward Port: `25565`
- TCP/UDP Forwarding: Both

## Backup & Restore

### Backup

```bash
# Stop NPM
docker stop nginx-proxy-manager

# Backup data
tar czf npm-backup.tar.gz ./data ./letsencrypt

# Start NPM
docker start nginx-proxy-manager
```

### Restore

```bash
docker stop nginx-proxy-manager
tar xzf npm-backup.tar.gz
docker start nginx-proxy-manager
```

## Integration with Docker

### Internal Docker Network

```yaml
networks:
  proxy:
    external: true

services:
  npm:
    networks:
      - proxy

  jellyfin:
    networks:
      - proxy
```

Forward to container name:
- Hostname: `jellyfin`
- Port: `8096`

## Troubleshooting

### SSL Certificate Failed

1. Check DNS propagation
2. Verify API credentials
3. Check rate limits (Let's Encrypt)
4. Review NPM logs: `docker logs nginx-proxy-manager`

### 502 Bad Gateway

1. Verify upstream service is running
2. Check hostname/IP is correct
3. Verify port is correct
4. Check network connectivity

### Websockets Not Working

1. Enable "Websockets Support" on host
2. Add custom config:
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

### Access Logs

Settings → Default Site → Show Logs

Or check:
```bash
docker logs -f nginx-proxy-manager
```

## Migrations

### From Nginx

1. Create proxy hosts for each server block
2. Request SSL certificates
3. Update DNS to point to NPM
4. Test each service

### From Traefik

1. Map labels to proxy host settings
2. Recreate access rules as access lists
3. Import or re-request SSL certificates

## Best Practices

1. **Use Docker network** - container names as hostnames
2. **Request wildcard cert** - one cert for all subdomains
3. **Enable HTTP/2** - better performance
4. **Set up access lists** - protect admin panels
5. **Regular backups** - especially `/data` and `/letsencrypt`
6. **Use custom configs sparingly** - harder to maintain
7. **Monitor certificate expiry** - auto-renewal should work
