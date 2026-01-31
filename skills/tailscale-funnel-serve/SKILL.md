# tailscale-funnel-serve

Tailscale Serve and Funnel - expose services to tailnet or public internet

## Source

- Serve: https://tailscale.com/kb/1242/tailscale-serve
- Funnel: https://tailscale.com/kb/1311/tailscale-funnel

## Overview

| Feature | Serve | Funnel |
|---------|-------|--------|
| Audience | Tailnet only | Public internet |
| URL | `https://device.tail12345.ts.net` | Same URL, publicly accessible |
| Auth | Tailscale identity | Anyone |
| HTTPS | Auto TLS cert | Auto TLS cert |
| Use case | Internal tools | Webhooks, demos, public APIs |

## Tailscale Serve

Expose local services to your tailnet with automatic HTTPS.

### Basic Usage

```bash
# Serve local port 3000 over HTTPS
tailscale serve 3000

# Serve with explicit localhost
tailscale serve localhost:3000

# Serve on specific path
tailscale serve --set-path=/api localhost:8080

# Serve static file
tailscale serve /path/to/index.html

# Serve directory
tailscale serve /path/to/files/

# Serve static text (debugging)
tailscale serve text:"Hello, Tailscale!"
```

### HTTPS (Default)
```bash
# Serve on HTTPS port 443 (default)
tailscale serve --https=443 localhost:3000

# Custom HTTPS port
tailscale serve --https=8443 localhost:3000
```

### HTTP Server
```bash
# Serve over HTTP (no TLS)
tailscale serve --http=80 localhost:3000
```

HTTP accessible via short MagicDNS: `http://my-device`

### TCP Forwarding
```bash
# Forward raw TCP
tailscale serve --tcp=2222 localhost:22

# TLS-terminated TCP
tailscale serve --tls-terminated-tcp=443 localhost:8080
```

### Background Mode
```bash
# Run persistently (survives reboots)
tailscale serve --bg localhost:3000

# Foreground (stops when terminal closes)
tailscale serve localhost:3000
```

### Self-Signed Certs
```bash
# Local service has self-signed cert
tailscale serve https+insecure://localhost:8443
```

### Path Routing
```bash
# Multiple services on different paths
tailscale serve --set-path=/app localhost:3000
tailscale serve --set-path=/api localhost:8080
tailscale serve --set-path=/docs /var/www/docs/
```

### Check Status
```bash
tailscale serve status

# JSON output
tailscale serve status --json
```

### Stop/Reset
```bash
# Turn off specific serve
tailscale serve --https=443 off

# Reset all serve config
tailscale serve reset
```

## Tailscale Funnel

Expose services to the public internet.

### Enable Funnel

First, enable in Admin Console:
1. Go to DNS settings
2. Enable HTTPS Certificates
3. Enable Funnel

### Basic Usage

```bash
# Expose to public internet
tailscale funnel 3000

# With path
tailscale funnel --set-path=/webhook localhost:8080
```

### Allowed Ports
Funnel only works on these ports:
- 443 (default)
- 8443
- 10000

```bash
tailscale funnel --https=443 localhost:3000
tailscale funnel --https=8443 localhost:3000
tailscale funnel --https=10000 localhost:3000
```

### TCP Forwarding
```bash
# Raw TCP (port 443, 8443, or 10000 only)
tailscale funnel --tcp=443 localhost:22

# TLS-terminated TCP
tailscale funnel --tls-terminated-tcp=443 localhost:8080
```

### Background Mode
```bash
tailscale funnel --bg localhost:3000
```

### PROXY Protocol
Preserve client IP addresses:
```bash
# Version 2 recommended
tailscale funnel --proxy-protocol=2 localhost:3000
```

Backend receives original client IP via PROXY protocol header.

### Stop Funnel
```bash
tailscale funnel --https=443 off
tailscale funnel reset
```

## JSON Configuration

For Docker/CI, use JSON config via `TS_SERVE_CONFIG`:

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "my-device.tail12345.ts.net:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:3000"
        },
        "/api": {
          "Proxy": "http://127.0.0.1:8080"
        },
        "/static": {
          "Path": "/var/www/static"
        }
      }
    }
  },
  "AllowFunnel": {
    "my-device.tail12345.ts.net:443": true
  }
}
```

Export current config:
```bash
tailscale serve status --json > serve-config.json
```

## Common Use Cases

### Local Development
```bash
# Share dev server with team
tailscale serve 3000
# Team accesses: https://my-laptop.tail12345.ts.net
```

### Webhook Testing
```bash
# Expose webhook endpoint publicly
tailscale funnel --set-path=/webhook localhost:8080
# Give URL to external service
```

### Demo to Client
```bash
# Show WIP to external client
tailscale funnel 3000
# Share public URL
```

### Internal Tool
```bash
# Internal dashboard accessible to team
tailscale serve --bg localhost:3000
```

### Home Services
```bash
# Home Assistant, Plex, etc.
tailscale serve --bg localhost:8123
```

## ACL Requirements

For Funnel, enable via nodeAttrs:

```json
{
  "nodeAttrs": [
    {
      "target": ["autogroup:member"],
      "attr": ["funnel"]
    }
  ]
}
```

Or restrict to specific tags:
```json
{
  "nodeAttrs": [
    {
      "target": ["tag:public"],
      "attr": ["funnel"]
    }
  ]
}
```

## Platform Notes

### macOS
- File serving requires open-source variant (not App Store version)
- Port serving works on all variants

### Docker
```yaml
environment:
  - TS_SERVE_CONFIG=/config/serve.json
volumes:
  - ./serve.json:/config/serve.json:ro
```

Mount config as directory for live updates.

## Troubleshooting

### HTTPS Not Working
```bash
# Check HTTPS certificates enabled in DNS settings
# Verify device has valid MagicDNS name
tailscale status
```

### Funnel Not Accessible
1. Verify Funnel enabled in Admin Console
2. Check nodeAttrs ACL allows funnel
3. Confirm using allowed port (443, 8443, 10000)
4. Test: `curl https://your-device.tail12345.ts.net`

### Service Not Responding
```bash
# Check serve status
tailscale serve status

# Verify local service is running
curl localhost:3000

# Check serve config
tailscale serve status --json
```

### Certificate Errors
```bash
# Certificates auto-provisioned
# May take a minute on first serve
# Check DNS settings in Admin Console
```

## Best Practices

1. **Use Serve for internal** - Keep internal tools off public internet
2. **Funnel for webhooks** - External integrations that need inbound
3. **Background mode for persistence** - Use `--bg` for long-running
4. **Path routing** - Multiple services on one hostname
5. **PROXY protocol** - When you need client IPs
6. **Restrict Funnel via ACL** - Only enable where needed

## Comparison

| Feature | Serve | Funnel | ngrok/Cloudflare Tunnel |
|---------|-------|--------|-------------------------|
| Audience | Tailnet | Public | Public |
| Auth | Tailscale | None | Varies |
| Setup | One command | One command | More config |
| Custom domain | ts.net only | ts.net only | Yes |
| Pricing | Included | Included | May cost |
| Integration | Native | Native | Separate tool |
