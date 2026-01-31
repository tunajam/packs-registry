# Pi-hole DNS

Network-wide ad blocking and DNS management.

## When to Use

- Block ads for all devices on network
- Custom DNS filtering
- Monitor DNS queries
- Replace router DHCP

## Docker Setup

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    hostname: pihole
    environment:
      - TZ=America/Los_Angeles
      - WEBPASSWORD=changeme
      - FTLCONF_LOCAL_IPV4=192.168.1.10
      - PIHOLE_DNS_=1.1.1.1;1.0.0.1
      - DNSSEC=true
      - QUERY_LOGGING=true
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"    # Web interface
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```

### With DHCP Enabled

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    network_mode: host
    environment:
      - TZ=America/Los_Angeles
      - WEBPASSWORD=changeme
      - FTLCONF_LOCAL_IPV4=192.168.1.10
      - PIHOLE_DNS_=1.1.1.1;1.0.0.1
      - DHCP_ACTIVE=true
      - DHCP_START=192.168.1.100
      - DHCP_END=192.168.1.200
      - DHCP_ROUTER=192.168.1.1
      - PIHOLE_DOMAIN=home.local
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```

## Router Configuration

### Point DNS to Pi-hole

1. **Router DHCP settings** → Set DNS server to Pi-hole IP
2. **Or** set Pi-hole as DHCP server → disable router DHCP

### Static IP for Pi-hole

Always give Pi-hole a static IP (e.g., 192.168.1.10).

## Custom Blocklists

### Add via Web UI

Settings → Adlists → Add:

```
# Popular lists
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://raw.githubusercontent.com/PolishFiltersTeam/KADhosts/master/KADhosts.txt
https://raw.githubusercontent.com/FadeMind/hosts.extras/master/add.Spam/hosts
https://v.firebog.net/hosts/Easyprivacy.txt
https://v.firebog.net/hosts/Prigent-Crypto.txt
```

### Firebog Ticked Lists (Curated)

```bash
# Download all "ticked" lists
curl -sSL https://v.firebog.net/hosts/lists.php?type=tick | xargs -I{} pihole -a adlist add {}
```

### Update Lists

```bash
pihole -g  # Update gravity (blocklists)
```

## Whitelist & Blacklist

### Whitelist (Allow)

```bash
pihole -w example.com
pihole -w s.youtube.com  # YouTube history
pihole -w cdn.embedly.com  # Twitter images
```

### Blacklist (Block)

```bash
pihole -b ads.example.com
pihole --regex ".*\.malware\.com$"  # Regex block
```

### Bulk Whitelist

```bash
# Common false positives
pihole -w \
  s.youtube.com \
  video-stats.l.google.com \
  www.googleadservices.com \
  clients4.google.com \
  clients2.google.com \
  s.gateway.messenger.live.com
```

## Local DNS Records

### Via Web UI

Local DNS → DNS Records:
- Domain: server.home.local
- IP: 192.168.1.50

### Via File

`/etc/pihole/custom.list`:
```
192.168.1.10 pihole.home.local
192.168.1.20 nas.home.local
192.168.1.30 proxmox.home.local
192.168.1.50 jellyfin.home.local
```

### Wildcard DNS

`/etc/dnsmasq.d/02-custom.conf`:
```
address=/home.local/192.168.1.10
```

This routes `*.home.local` to your reverse proxy.

## CNAME Records

`/etc/dnsmasq.d/05-cname.conf`:
```
cname=jellyfin.home.local,server.home.local
cname=sonarr.home.local,server.home.local
```

## Conditional Forwarding

For local hostname resolution via your router:

Settings → DNS:
- Enable Conditional Forwarding
- Local network: 192.168.1.0/24
- Router IP: 192.168.1.1
- Local domain: home.local

## Unbound (Recursive DNS)

Run your own recursive DNS instead of using upstream:

```yaml
services:
  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    volumes:
      - ./unbound:/opt/unbound/etc/unbound
    ports:
      - "5335:53/tcp"
      - "5335:53/udp"
    restart: unless-stopped

  pihole:
    # ...
    environment:
      - PIHOLE_DNS_=unbound#5335
    depends_on:
      - unbound
```

## Monitoring

### CLI Stats

```bash
pihole -c  # Chronometer (live stats)
pihole status
pihole -t  # Tail log
```

### API

```bash
# Summary stats
curl "http://pihole.local/admin/api.php?summary"

# Top blocked domains
curl "http://pihole.local/admin/api.php?topItems&auth=TOKEN"
```

## Troubleshooting

### Check DNS Resolution

```bash
# From any device
nslookup google.com 192.168.1.10

# Should show Pi-hole as server
dig @192.168.1.10 google.com

# Check if domain is blocked
pihole -q ads.example.com
```

### Debug Mode

```bash
pihole debug
```

### Common Issues

**Port 53 in use:**
```bash
# Check what's using port 53
sudo lsof -i :53

# Disable systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```

**Docker DNS conflict:**
```yaml
# Set static DNS in container
dns:
  - 127.0.0.1
  - 1.1.1.1
```

## Group Management

### Create Groups

Settings → Groups → Add group:
- kids (restricted)
- adults (less restricted)
- iot (heavy filtering)

### Assign Clients to Groups

Settings → Clients → Add client:
- 192.168.1.100 → kids group

### Assign Lists to Groups

Settings → Adlists → Edit → Select groups

## Useful Commands

```bash
# Update Pi-hole
pihole -up

# Update gravity (blocklists)
pihole -g

# Restart DNS
pihole restartdns

# Disable temporarily
pihole disable 5m  # 5 minutes

# Enable
pihole enable

# Query log
pihole -t

# Flush logs
pihole flush
```

## Best Practices

1. **Static IP required** - Pi-hole must have a fixed address
2. **Secondary DNS** - run two instances for redundancy
3. **Start conservative** - add blocklists gradually
4. **Whitelist carefully** - some blocks break sites
5. **Enable query logging** - helps debug issues
6. **Regular updates** - `pihole -up` and `pihole -g`
7. **Backup config** - `/etc/pihole` and `/etc/dnsmasq.d`
