# SSL/TLS Certificates

Free HTTPS for your homelab with Let's Encrypt.

## When to Use

- Enable HTTPS for all services
- Secure internal services
- Wildcard certificates for subdomains
- Auto-renewal without downtime

## Challenge Types

| Challenge | Port Required | Wildcard | Best For |
|-----------|---------------|----------|----------|
| HTTP-01 | 80 (public) | No | Public servers |
| DNS-01 | None | Yes | Internal, wildcard |
| TLS-ALPN-01 | 443 (public) | No | Special cases |

**Recommendation:** Use DNS-01 for homelab (works behind firewall, supports wildcards).

## acme.sh (Standalone)

### Install

```bash
curl https://get.acme.sh | sh
source ~/.bashrc
```

### DNS Providers

Set credentials in environment:

**Cloudflare:**
```bash
export CF_Token="your-api-token"
export CF_Zone_ID="your-zone-id"
```

**Vercel:**
```bash
export VERCEL_TOKEN="your-token"
```

**Route53:**
```bash
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
```

### Issue Certificate

```bash
# Single domain
acme.sh --issue --dns dns_cf -d example.com

# Wildcard + root
acme.sh --issue --dns dns_cf -d example.com -d '*.example.com'

# Multiple wildcards
acme.sh --issue --dns dns_cf \
  -d example.com \
  -d '*.example.com' \
  -d '*.home.example.com'
```

### Install Certificate

```bash
# For nginx
acme.sh --install-cert -d example.com \
  --key-file /etc/nginx/ssl/key.pem \
  --fullchain-file /etc/nginx/ssl/cert.pem \
  --reloadcmd "systemctl reload nginx"

# For generic use
acme.sh --install-cert -d example.com \
  --cert-file /path/to/cert.pem \
  --key-file /path/to/key.pem \
  --fullchain-file /path/to/fullchain.pem \
  --ca-file /path/to/ca.pem
```

### Auto-Renewal

acme.sh sets up a cron job automatically:
```bash
# Check cron
crontab -l | grep acme

# Manual renewal
acme.sh --renew -d example.com --force
```

## Certbot

### Install

```bash
# Ubuntu/Debian
apt install certbot

# With nginx plugin
apt install python3-certbot-nginx
```

### HTTP Challenge (Public Server)

```bash
# Standalone (stops web server)
certbot certonly --standalone -d example.com

# With nginx plugin (no downtime)
certbot --nginx -d example.com
```

### DNS Challenge

```bash
# Manual (interactive)
certbot certonly --manual --preferred-challenges dns -d '*.example.com'

# With DNS plugin (automated)
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.cloudflare.ini \
  -d example.com -d '*.example.com'
```

`/root/.cloudflare.ini`:
```ini
dns_cloudflare_api_token = your-token
```

### Renewal

```bash
# Test renewal
certbot renew --dry-run

# Force renewal
certbot renew --force-renewal
```

## Traefik (Automatic)

```yaml
# traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
```

```yaml
# docker-compose.yml
services:
  traefik:
    environment:
      - CF_DNS_API_TOKEN=your-token
```

## Caddy (Automatic)

```caddyfile
# Wildcard with DNS challenge
*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
}
```

## Internal/Tailscale Certificates

### Option 1: DNS Challenge for Real Domain

Point internal subdomain to internal IP:
```
internal.example.com -> 192.168.1.10
```

Use DNS challenge:
```bash
acme.sh --issue --dns dns_cf -d internal.example.com
```

### Option 2: Tailscale HTTPS

```bash
# Enable HTTPS on Tailscale
tailscale serve https / http://localhost:8080
```

### Option 3: Self-Signed (Last Resort)

```bash
# Generate self-signed cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/selfsigned.key \
  -out /etc/ssl/certs/selfsigned.crt \
  -subj "/CN=internal.local"
```

## Certificate Formats

### Convert Formats

```bash
# PEM to PKCS12
openssl pkcs12 -export -out cert.pfx \
  -inkey key.pem -in cert.pem -certfile chain.pem

# PKCS12 to PEM
openssl pkcs12 -in cert.pfx -out cert.pem -nodes

# PEM to DER
openssl x509 -outform der -in cert.pem -out cert.der
```

### View Certificate

```bash
# View local cert
openssl x509 -in cert.pem -text -noout

# View remote cert
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -text -noout

# Check expiry
openssl x509 -enddate -noout -in cert.pem
```

## Monitoring Expiry

### Script Check

```bash
#!/bin/bash
DOMAIN="example.com"
DAYS_WARNING=30

EXPIRY=$(echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | \
  openssl x509 -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))

if [ $DAYS_LEFT -lt $DAYS_WARNING ]; then
  echo "WARNING: $DOMAIN expires in $DAYS_LEFT days"
fi
```

### Prometheus Blackbox Exporter

```yaml
- job_name: 'ssl'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://example.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

Alert rule:
```yaml
- alert: SSLCertExpiring
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 14
  labels:
    severity: warning
  annotations:
    summary: "SSL cert expires in {{ $value | humanizeDuration }}"
```

## Troubleshooting

### DNS Propagation

```bash
# Check TXT record propagation
dig _acme-challenge.example.com TXT +short

# Use specific DNS server
dig @8.8.8.8 _acme-challenge.example.com TXT +short
```

### Rate Limits

Let's Encrypt limits:
- 50 certificates per domain per week
- 5 duplicate certificates per week
- 300 new orders per account per 3 hours

**Tip:** Use staging for testing:
```bash
acme.sh --issue --staging --dns dns_cf -d example.com
```

### Permission Issues

```bash
# Fix acme.sh permissions
chmod 600 /root/.acme.sh/*.key
chmod 644 /root/.acme.sh/*.cer

# Fix certbot permissions
chmod 600 /etc/letsencrypt/live/*/privkey.pem
```

## Best Practices

1. **Use DNS challenge** - works everywhere, supports wildcards
2. **Automate renewal** - certs expire in 90 days
3. **Monitor expiry** - alert before expiration
4. **Use staging first** - avoid rate limits during testing
5. **Secure private keys** - chmod 600, restricted access
6. **Backup certificates** - especially for complex setups
7. **Use SAN certs** - multiple domains in one cert
