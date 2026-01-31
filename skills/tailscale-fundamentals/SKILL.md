# tailscale-fundamentals

Core Tailscale concepts - tailnets, MagicDNS, device management, and basic setup

## Source

Tailscale official documentation: https://tailscale.com/kb

## What is Tailscale?

Tailscale is a mesh VPN built on WireGuard that creates a secure network (tailnet) between your devices. Unlike traditional VPNs, Tailscale:

- Creates direct peer-to-peer connections between devices
- Uses WireGuard for encryption (fast, modern, audited)
- Requires no server infrastructure to manage
- Works through NAT and firewalls automatically

## Core Concepts

### Tailnet
Your private Tailscale network. Each organization/account has one tailnet containing all devices and users.

### MagicDNS
Automatic DNS for your tailnet. Devices get hostnames like `my-laptop.tail1234.ts.net`.

```bash
# Enable MagicDNS
tailscale up --accept-dns

# Access devices by name
ssh my-server  # Instead of IP
```

### Tailscale IP Addresses
Every device gets a stable IP in the 100.x.y.z range (CGNAT space). This IP stays the same regardless of physical network.

## Essential CLI Commands

```bash
# Check status
tailscale status

# Get your Tailscale IP
tailscale ip

# Connect/authenticate
tailscale up

# Disconnect (stays authenticated)
tailscale down

# Full logout
tailscale logout

# Check network connectivity
tailscale netcheck

# Ping another device
tailscale ping <hostname-or-ip>

# Debug connectivity
tailscale debug
```

## Device Management

### View Devices
```bash
# List all devices in your tailnet
tailscale status

# Detailed device info (JSON)
tailscale status --json
```

### Machine Names
Devices can be renamed in the admin console or via API. MagicDNS names update automatically.

### Key Expiry
- Node keys expire by default (90 days for personal, configurable for teams)
- Tagged devices can disable key expiry
- Re-authenticate before expiry to maintain connectivity

```bash
# Check key expiry
tailscale status --json | jq '.Self.KeyExpiry'
```

## Authentication

### Auth Keys
Pre-generate keys for automated device registration:

```bash
# Use an auth key
tailscale up --authkey=tskey-auth-xxxxx

# Ephemeral key (device removed when it goes offline)
tailscale up --authkey=tskey-auth-xxxxx?ephemeral=true
```

Auth key types:
- **Reusable**: Can register multiple devices
- **Ephemeral**: Devices auto-removed when offline
- **Pre-approved**: Skip admin approval
- **Tagged**: Apply ACL tags automatically

### OAuth Clients
For programmatic access to the Tailscale API and automated device registration.

## Network Configuration

### Accept Routes
Enable access to subnet routers:
```bash
tailscale up --accept-routes
```

### Advertise Routes
Share a subnet with your tailnet:
```bash
tailscale up --advertise-routes=192.168.1.0/24
```

### Hostname
Set a custom hostname:
```bash
tailscale up --hostname=my-custom-name
```

## Common Patterns

### Personal Use
1. Install Tailscale on all your devices
2. Sign in with same account
3. Access any device from anywhere

### Team/Org Use
1. Set up SSO with identity provider
2. Define ACLs for access control
3. Use tags for server roles
4. Create groups for team access

### Server Access
```bash
# On the server
tailscale up --ssh  # Enable Tailscale SSH

# From client
ssh root@my-server  # Uses Tailscale auth
```

## Troubleshooting

### Can't Connect?
```bash
# Check network
tailscale netcheck

# Look for relay vs direct
tailscale status

# Debug specific peer
tailscale ping --verbose <peer>
```

### DNS Issues?
```bash
# Check DNS config
tailscale status --json | jq '.DNS'

# Force DNS acceptance
tailscale up --accept-dns
```

### Behind Corporate Firewall?
Tailscale uses DERP relays when direct connections fail. Check:
- Port 41641 UDP (preferred)
- Port 443 TCP (fallback)

## Best Practices

1. **Use tags for servers** - Decouple access from user accounts
2. **Enable MagicDNS** - Makes device access intuitive
3. **Set up ACLs early** - Default allows all; lock down before production
4. **Use ephemeral keys for CI/CD** - Auto-cleanup of temporary devices
5. **Monitor key expiry** - Set up alerts for upcoming expirations

## Admin Console

Access at https://login.tailscale.com/admin:
- View all devices
- Manage users and roles
- Configure ACLs
- Generate auth keys
- View network logs
