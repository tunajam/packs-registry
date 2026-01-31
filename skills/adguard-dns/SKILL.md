# AdGuard Home DNS

Network-wide ad blocking with modern features.

## When to Use

- Block ads network-wide
- Need DoH/DoT encrypted DNS
- Want parental controls
- Prefer modern UI over Pi-hole

## AdGuard vs Pi-hole

| Feature | AdGuard | Pi-hole |
|---------|---------|---------|
| Encrypted DNS (DoH/DoT) | Native | Requires extra setup |
| Parental controls | Built-in | Via blocklists |
| Per-client settings | Yes | Via groups |
| DHCP server | Yes | Yes |
| UI | Modern | Classic |

## Docker Setup

```yaml
services:
  adguardhome:
    image: adguard/adguardhome
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"      # Initial setup only
      - "443:443/tcp"    # HTTPS UI
      - "3000:3000/tcp"  # Setup wizard
      - "853:853/tcp"    # DNS-over-TLS
      - "784:784/udp"    # DNS-over-QUIC
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
```

### With DHCP

```yaml
services:
  adguardhome:
    image: adguard/adguardhome
    container_name: adguardhome
    network_mode: host
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
    restart: unless-stopped
```

## Initial Setup

1. Navigate to `http://your-ip:3000`
2. Configure admin interface (port 80 or custom)
3. Set DNS port (53)
4. Create admin credentials
5. Configure upstream DNS

## Upstream DNS

### Encrypted Upstream (Recommended)

Settings → DNS settings → Upstream DNS:
```
# Cloudflare DoH
https://cloudflare-dns.com/dns-query

# Quad9 DoT
tls://dns.quad9.net

# Google DoH
https://dns.google/dns-query
```

### Parallel Requests

Enable "Parallel requests" for fastest response.

### Bootstrap DNS

For resolving DoH/DoT server names:
```
1.1.1.1
8.8.8.8
```

## Blocklists

### Add Blocklist

Filters → DNS Blocklists → Add:
```
# AdGuard Default
https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt

# OISD (comprehensive)
https://big.oisd.nl

# Steven Black hosts
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts

# Energized Ultimate
https://block.energized.pro/ultimate/formats/hosts
```

### Whitelist

Filters → DNS Allowlist:
```
# Common false positives
s.youtube.com
video-stats.l.google.com
||spclient.wg.spotify.com
||redirector.gvt1.com
```

## DNS Rewrites (Local DNS)

Filters → DNS rewrites → Add:
```
Domain: jellyfin.home.local
Answer: 192.168.1.50

Domain: *.home.local
Answer: 192.168.1.10  # Reverse proxy
```

## Per-Client Settings

### Define Clients

Settings → Client settings → Add client:
- Name: Kids-iPad
- Identifiers: 192.168.1.100 or MAC
- Tags: child

### Client-Specific Filtering

Per client:
- Use global settings
- Block specific services
- Custom blocklists
- Parental controls

## Parental Controls

Settings → General settings:
- Enable parental control
- Block adult sites

Or per-client:
- Enable Safe Search (forces safe mode on search engines)
- Block specific services

### Blocked Services

Quick toggles for:
- Social media (TikTok, Instagram, etc.)
- Gaming (Steam, Epic, etc.)
- Streaming (Netflix, YouTube)
- Adult content

## Encrypted DNS Server

### DNS-over-HTTPS (DoH)

Settings → Encryption settings:
- Enable encryption
- Server name: `dns.example.com`
- Certificate: Paste or path
- Private key: Paste or path

Access: `https://dns.example.com/dns-query`

### DNS-over-TLS (DoT)

Same certificate setup.
Access: `tls://dns.example.com`

### DNS-over-QUIC (DoQ)

Port 784/UDP automatically enabled.
Access: `quic://dns.example.com`

## DHCP Server

Settings → DHCP settings:
- Enable DHCP
- Interface: eth0
- Gateway: 192.168.1.1
- Subnet mask: 255.255.255.0
- Range: 192.168.1.100 - 192.168.1.200
- Lease duration: 24h

### Static Leases

DHCP → Static leases → Add:
- MAC: aa:bb:cc:dd:ee:ff
- IP: 192.168.1.50
- Hostname: server

## Query Log

### View Queries

Query Log:
- Filter by client
- Filter by status (blocked/allowed)
- Block/unblock domains directly

### Retention

Settings → General settings:
- Query log retention: 90 days (default)
- Anonymize client IPs: Optional for privacy

## Backup & Restore

### Backup Config

```bash
# Config file
cp ./conf/AdGuardHome.yaml ./backup/

# Or backup entire conf directory
tar czf adguard-backup.tar.gz ./conf
```

### Settings Export

Settings → General settings → Backup and restore

## API Access

```bash
# Get filtering status
curl -u admin:password "http://adguard:3000/control/status"

# Enable/disable filtering
curl -u admin:password -X POST "http://adguard:3000/control/filtering/disable"
```

## Monitoring

### Statistics

Dashboard shows:
- Total queries
- Blocked queries
- Top clients
- Top blocked domains

### Prometheus Metrics

Not built-in, but exporters available:
```yaml
adguard-exporter:
  image: ebrianne/adguard-exporter
  environment:
    - ADGUARD_PROTOCOL=http
    - ADGUARD_HOSTNAME=adguardhome
    - ADGUARD_USERNAME=admin
    - ADGUARD_PASSWORD=password
  ports:
    - "9617:9617"
```

## Troubleshooting

### Port 53 in Use

```bash
# Check what's using port 53
sudo lsof -i :53

# Disable systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```

### Queries Not Being Blocked

1. Verify DNS is pointing to AdGuard
2. Check blocklists are enabled
3. Review custom rules
4. Check client-specific settings

### High Memory Usage

Reduce query log retention or disable logging.

## Best Practices

1. **Use encrypted upstream** - DoH or DoT
2. **Start with fewer blocklists** - add as needed
3. **Monitor false positives** - whitelist carefully
4. **Per-client configs** - different rules for kids
5. **Backup config** - before major changes
6. **Enable query logging** - helps troubleshooting
7. **Set up failover DNS** - secondary AdGuard instance
