# tailscale-subnet-routing

Subnet routers, exit nodes, and network routing patterns in Tailscale

## Source

Tailscale subnet routing: https://tailscale.com/kb/1019/subnets

## Overview

Subnet routers let Tailscale devices access networks that can't run Tailscale directly. Exit nodes route all internet traffic through a specific device.

## Subnet Routers

### What They Do
- Bridge Tailscale network to physical subnets
- Enable access to devices without Tailscale installed
- Connect cloud VPCs, office networks, IoT devices

### When to Use
- Legacy devices (printers, cameras, NAS)
- Cloud managed services (RDS, Cloud SQL)
- Large networks where per-device install is impractical
- Gradual Tailscale migration

### Setup (Linux)

```bash
# 1. Enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# 2. If using firewalld, allow masquerading
sudo firewall-cmd --permanent --add-masquerade

# 3. Advertise routes
sudo tailscale up --advertise-routes=192.168.1.0/24,10.0.0.0/8

# 4. Approve routes in admin console (or use autoApprovers)
```

### Setup (macOS)
```bash
# Enable and advertise routes
sudo tailscale up --advertise-routes=192.168.1.0/24
```

### Client Usage
```bash
# Accept routes from subnet routers
tailscale up --accept-routes
```

## Exit Nodes

### What They Do
- Route ALL internet traffic through a device
- Acts like a traditional VPN
- Useful for untrusted networks (coffee shops, airports)

### Setup Exit Node

```bash
# On the exit node device
tailscale up --advertise-exit-node

# Approve in admin console or via autoApprovers
```

### Use Exit Node

```bash
# From client device
tailscale up --exit-node=<exit-node-hostname>

# Or by IP
tailscale up --exit-node=100.x.y.z

# Allow local network access while using exit node
tailscale up --exit-node=my-exit --exit-node-allow-lan-access

# Stop using exit node
tailscale up --exit-node=
```

### Suggested Exit Nodes
Tailscale can auto-select the best exit node:
```bash
tailscale up --exit-node=auto
```

## High Availability

### Multiple Subnet Routers
Advertise the same routes from multiple devices for redundancy:

```bash
# On router-1
tailscale up --advertise-routes=192.168.1.0/24

# On router-2  
tailscale up --advertise-routes=192.168.1.0/24
```

Tailscale automatically fails over if one router goes offline.

**Warning**: Don't use `--accept-routes` on HA subnet routers sharing the same routes—it creates inefficient routing loops.

## SNAT (Source NAT)

By default, subnet routers masquerade traffic (SNAT enabled). The subnet sees traffic as coming from the router.

### Disable SNAT
To preserve original source IPs:

```bash
# Linux only
tailscale up --advertise-routes=192.168.1.0/24 --snat-subnet-routes=false
```

When SNAT is disabled, you must add return routes on subnet devices:
- Route: `100.64.0.0/10` → subnet router's LAN IP

## ACL Configuration

### Allow Subnet Access
```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["192.168.1.0/24:*"]
    }
  ]
}
```

### Auto-Approve Routes
```json
{
  "autoApprovers": {
    "routes": {
      "192.168.1.0/24": ["tag:router"],
      "10.0.0.0/8": ["group:devops"]
    },
    "exitNode": ["tag:exit"]
  }
}
```

### Subnet-to-Subnet Communication
With SNAT disabled, subnets can communicate:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["192.168.1.0/24"],
      "dst": ["192.168.2.0/24:*"]
    },
    {
      "action": "accept",
      "src": ["192.168.2.0/24"],
      "dst": ["192.168.1.0/24:*"]
    }
  ]
}
```

## Overlapping Routes

Tailscale uses longest-prefix matching:

```
Router A: 10.0.0.0/16
Router B: 10.0.0.0/24

Traffic to 10.0.0.5 → Router B (more specific /24)
Traffic to 10.0.1.5 → Router A (only matches /16)
```

Use case: Route specific subnets through dedicated routers.

## DNS Configuration

### Split DNS
Route specific domains to internal DNS servers:

1. Go to Admin Console → DNS
2. Add nameserver for your domain
3. Restrict to specific search domain

Example: `*.internal.company.com` → `192.168.1.53`

## Common Patterns

### VPC Access
```bash
# On VM in VPC
tailscale up --advertise-routes=10.0.0.0/16 --advertise-tags=tag:vpc-router
```

### Home Network Access
```bash
# On home server/pi
tailscale up --advertise-routes=192.168.1.0/24
```

### Site-to-Site
Connect multiple office networks:
```bash
# Office A
tailscale up --advertise-routes=10.1.0.0/24

# Office B
tailscale up --advertise-routes=10.2.0.0/24

# Both offices can now communicate
```

### Exit Node for Privacy
```bash
# On VPS in preferred location
tailscale up --advertise-exit-node

# On laptop
tailscale up --exit-node=my-vps
```

## Troubleshooting

### Routes Not Working?
```bash
# Check if routes are enabled
tailscale status

# Verify IP forwarding
sysctl net.ipv4.ip_forward

# Check advertised routes
tailscale status --json | jq '.Self.AllowedIPs'
```

### Can't Reach Subnet?
1. Verify `--accept-routes` on client
2. Check ACLs allow access to subnet
3. Verify subnet router is online
4. Check firewall rules on subnet router

### Performance Issues?
- Prefer direct connections over relayed
- Check `tailscale status` for "relay" indicators
- Consider placing routers closer to users
- Enable userspace networking only when needed

## Best Practices

1. **Use tags for routers** - `tag:router`, `tag:exit`
2. **Auto-approve routes** - Reduce manual approval burden
3. **Deploy HA** - Multiple routers for critical subnets
4. **Monitor router health** - Alert on offline routers
5. **Document subnets** - Keep network topology current
6. **Restrict exit nodes** - Not everyone needs full internet routing

## Comparison

| Feature | Subnet Router | Exit Node |
|---------|--------------|-----------|
| Purpose | Access specific networks | Route all internet |
| Routes | Specific CIDRs | 0.0.0.0/0, ::/0 |
| Use case | Internal resources | Privacy, geo-access |
| Setup | --advertise-routes | --advertise-exit-node |
