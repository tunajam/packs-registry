# WireGuard VPN

Modern, fast VPN for secure remote access to your homelab.

## When to Use

- Remote access to home network
- Secure connection from public WiFi
- Site-to-site VPN between locations
- Bypass geo-restrictions

## Why WireGuard?

- **Fast** - kernel-level implementation
- **Simple** - ~4000 lines of code
- **Modern crypto** - ChaCha20, Curve25519
- **Cross-platform** - Linux, macOS, Windows, iOS, Android

## Docker Setup (Recommended)

### Using linuxserver/wireguard

```yaml
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - SERVERURL=vpn.example.com  # Your domain or IP
      - SERVERPORT=51820
      - PEERS=phone,laptop,tablet   # Client names
      - PEERDNS=auto                # Use container's DNS
      - INTERNAL_SUBNET=10.13.13.0  # VPN subnet
      - ALLOWEDIPS=0.0.0.0/0        # Route all traffic
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules:ro
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

### Get Client Configs

```bash
# QR code for mobile
docker exec wireguard cat /config/peer_phone/peer_phone.png | base64 -d > phone-qr.png

# Config file
cat config/peer_laptop/peer_laptop.conf
```

## Manual Server Setup

### Install

```bash
# Ubuntu/Debian
apt install wireguard

# Enable IP forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### Generate Keys

```bash
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

### Server Config (/etc/wireguard/wg0.conf)

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Phone
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32

[Peer]
# Laptop
PublicKey = <another_client_public_key>
AllowedIPs = 10.0.0.3/32
```

### Client Config

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <client_private_key>
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0  # Route all traffic
PersistentKeepalive = 25
```

### Start Server

```bash
wg-quick up wg0
systemctl enable wg-quick@wg0
```

## Configuration Options

### Split Tunnel (Only Route Homelab)

```ini
# Client config - only route local network
[Peer]
AllowedIPs = 192.168.1.0/24, 10.0.0.0/24
```

### Full Tunnel (Route All Traffic)

```ini
# Client config - route everything through VPN
[Peer]
AllowedIPs = 0.0.0.0/0, ::/0
```

### Site-to-Site VPN

**Site A (Server):**
```ini
[Interface]
Address = 10.0.0.1/24
PrivateKey = <site_a_private>
ListenPort = 51820

[Peer]
PublicKey = <site_b_public>
AllowedIPs = 10.0.0.2/32, 192.168.2.0/24
```

**Site B (Client):**
```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <site_b_private>

[Peer]
PublicKey = <site_a_public>
Endpoint = site-a.example.com:51820
AllowedIPs = 10.0.0.1/32, 192.168.1.0/24
PersistentKeepalive = 25
```

## Pi-hole as DNS

```yaml
services:
  wireguard:
    environment:
      - PEERDNS=192.168.1.10  # Pi-hole IP
```

Or in manual config:
```ini
[Interface]
DNS = 192.168.1.10
```

## Firewall Rules

### UFW

```bash
ufw allow 51820/udp
ufw route allow in on wg0 out on eth0
```

### iptables

```bash
# Allow WireGuard traffic
iptables -A INPUT -p udp --dport 51820 -j ACCEPT

# Forward traffic
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -j ACCEPT

# NAT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Monitoring

### Check Status

```bash
wg show
wg show wg0
```

### Output

```
interface: wg0
  public key: <server_public_key>
  private key: (hidden)
  listening port: 51820

peer: <client_public_key>
  endpoint: 1.2.3.4:54321
  allowed ips: 10.0.0.2/32
  latest handshake: 45 seconds ago
  transfer: 123.45 MiB received, 67.89 MiB sent
```

## Troubleshooting

### Connection Issues

```bash
# Check if port is open
nc -zvu vpn.example.com 51820

# Check routing
ip route show table main

# Debug WireGuard
wg show wg0
journalctl -u wg-quick@wg0
```

### No Internet Through VPN

1. Check IP forwarding: `cat /proc/sys/net/ipv4/ip_forward` (should be 1)
2. Check NAT rules: `iptables -t nat -L POSTROUTING`
3. Check DNS: `nslookup google.com`

### Handshake Issues

- **Wrong public key** - double-check key on both ends
- **Firewall blocking** - ensure UDP 51820 is open
- **NAT issues** - add `PersistentKeepalive = 25`

## Mobile Setup

### iOS/Android

1. Install WireGuard app
2. Scan QR code or import .conf file
3. Toggle connection on/off

### Generate QR Code

```bash
qrencode -t ansiutf8 < peer.conf
```

## Best Practices

1. **Use DNS names** - easier than remembering IPs
2. **Different keys per device** - revoke individually
3. **Backup keys** - store securely offline
4. **Use PersistentKeepalive** - for NAT traversal
5. **Split tunnel** - route only what you need
6. **Update regularly** - WireGuard is actively developed
7. **Use fail2ban** - optional but adds security
