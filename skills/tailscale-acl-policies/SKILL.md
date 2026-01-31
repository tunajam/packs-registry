# tailscale-acl-policies

Tailscale ACL policy patterns - groups, tags, autoApprovers, and access control

## Source

Tailscale ACL documentation: https://tailscale.com/kb/1018/acls

## Overview

Tailscale uses a **deny-by-default** model. ACLs (Access Control Lists) define what traffic is allowed. Policies are written in HuJSON (Human JSON) format.

## Policy File Structure

```json
{
  // Groups of users
  "groups": {
    "group:engineering": ["alice@example.com", "bob@example.com"],
    "group:devops": ["carol@example.com"]
  },

  // Who can assign tags
  "tagOwners": {
    "tag:server": ["group:devops"],
    "tag:prod": ["group:devops"]
  },

  // Access rules
  "acls": [
    {"action": "accept", "src": ["group:engineering"], "dst": ["tag:server:*"]}
  ],

  // Auto-approve routes
  "autoApprovers": {
    "routes": {
      "192.168.1.0/24": ["tag:router"]
    },
    "exitNode": ["tag:exit"]
  },

  // SSH access
  "ssh": [
    {"action": "accept", "src": ["autogroup:member"], "dst": ["tag:server"], "users": ["autogroup:nonroot"]}
  ],

  // Named hosts
  "hosts": {
    "my-server": "100.100.100.1"
  },

  // Policy tests
  "tests": [
    {"src": "alice@example.com", "accept": ["tag:server:22"]}
  ]
}
```

## Autogroups

Built-in groups that update automatically:

| Autogroup | Description |
|-----------|-------------|
| `autogroup:member` | All tailnet members |
| `autogroup:admin` | Users with Admin role |
| `autogroup:owner` | Tailnet owner |
| `autogroup:tagged` | All tagged devices |
| `autogroup:self` | User's own devices |
| `autogroup:internet` | Exit node internet access |
| `autogroup:shared` | Devices shared to your tailnet |
| `autogroup:nonroot` | SSH: any non-root user |

## Tags

Tags assign identity to devices independent of users. Essential for servers and services.

### Define Tag Owners
```json
{
  "tagOwners": {
    "tag:webserver": ["group:devops", "alice@example.com"],
    "tag:database": ["group:devops"],
    "tag:ci": ["autogroup:admin"]
  }
}
```

### Apply Tags
```bash
# When starting Tailscale
tailscale up --advertise-tags=tag:webserver

# With auth key (auto-tagged)
tailscale up --authkey=tskey-auth-xxx
```

### Tag Benefits
- Access based on role, not user
- Key expiry disabled by default
- Ideal for servers, VMs, containers

## Common ACL Patterns

### Allow All (Default)
```json
{
  "acls": [
    {"action": "accept", "src": ["*"], "dst": ["*:*"]}
  ]
}
```

### Deny All
```json
{
  "acls": []
}
```

### Users Access Own Devices
```json
{
  "acls": [
    {"action": "accept", "src": ["autogroup:member"], "dst": ["autogroup:self:*"]}
  ]
}
```

### Group-Based Access
```json
{
  "groups": {
    "group:engineering": ["alice@example.com", "bob@example.com"],
    "group:devops": ["carol@example.com"]
  },
  "acls": [
    // Engineering can access dev servers
    {"action": "accept", "src": ["group:engineering"], "dst": ["tag:dev:*"]},
    // DevOps can access everything
    {"action": "accept", "src": ["group:devops"], "dst": ["tag:dev:*", "tag:prod:*"]}
  ]
}
```

### Restrict by Port
```json
{
  "acls": [
    // Only web ports
    {"action": "accept", "src": ["group:web-team"], "dst": ["tag:webserver:80,443"]},
    // SSH only for admins
    {"action": "accept", "src": ["group:devops"], "dst": ["tag:webserver:22"]}
  ]
}
```

### Restrict by Protocol
```json
{
  "acls": [
    // TCP only
    {"action": "accept", "src": ["*"], "dst": ["tag:server:*"], "proto": "tcp"},
    // UDP for specific use
    {"action": "accept", "src": ["*"], "dst": ["tag:dns:53"], "proto": "udp"}
  ]
}
```

## Auto Approvers

Automatically approve routes and exit nodes without admin intervention.

```json
{
  "autoApprovers": {
    "routes": {
      "10.0.0.0/8": ["tag:router", "group:devops"],
      "192.168.0.0/16": ["tag:home-router"]
    },
    "exitNode": ["tag:exit-node"]
  }
}
```

## SSH Policies

Control Tailscale SSH access:

```json
{
  "ssh": [
    // Users can SSH to their own devices
    {
      "action": "check",
      "src": ["autogroup:member"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot"]
    },
    // DevOps can SSH to servers as any user
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:server"],
      "users": ["*"]
    },
    // Engineering SSH with re-auth every 12h
    {
      "action": "check",
      "checkPeriod": "12h",
      "src": ["group:engineering"],
      "dst": ["tag:dev"],
      "users": ["ubuntu", "deploy"]
    }
  ]
}
```

### SSH Actions
- `accept`: Allow connection
- `check`: Require periodic re-authentication

## Named Hosts

Create aliases for IPs:

```json
{
  "hosts": {
    "prod-db": "100.100.1.1",
    "office-network": "192.168.1.0/24"
  },
  "acls": [
    {"action": "accept", "src": ["group:backend"], "dst": ["prod-db:5432"]}
  ]
}
```

## Policy Tests

Validate your ACLs won't break expected access:

```json
{
  "tests": [
    // Alice should access dev servers on port 22
    {"src": "alice@example.com", "accept": ["tag:dev:22"]},
    // Bob should NOT access prod
    {"src": "bob@example.com", "deny": ["tag:prod:22"]},
    // Tagged servers can reach database
    {"src": "tag:webserver", "accept": ["tag:database:5432"]}
  ]
}
```

## Production ACL Template

```json
{
  "groups": {
    "group:engineering": [],
    "group:devops": [],
    "group:security": []
  },
  
  "tagOwners": {
    "tag:server": ["group:devops"],
    "tag:prod": ["group:devops", "group:security"],
    "tag:dev": ["group:devops", "group:engineering"],
    "tag:exit": ["group:devops"]
  },

  "autoApprovers": {
    "routes": {
      "10.0.0.0/8": ["tag:router"]
    },
    "exitNode": ["tag:exit"]
  },

  "acls": [
    // Everyone accesses their own devices
    {"action": "accept", "src": ["autogroup:member"], "dst": ["autogroup:self:*"]},
    
    // Engineering accesses dev
    {"action": "accept", "src": ["group:engineering"], "dst": ["tag:dev:*"]},
    
    // DevOps accesses everything
    {"action": "accept", "src": ["group:devops"], "dst": ["*:*"]},
    
    // Security can audit all
    {"action": "accept", "src": ["group:security"], "dst": ["*:*"]},
    
    // Servers can talk to each other
    {"action": "accept", "src": ["tag:server"], "dst": ["tag:server:*"]}
  ],

  "ssh": [
    {"action": "check", "src": ["autogroup:member"], "dst": ["autogroup:self"], "users": ["autogroup:nonroot"]},
    {"action": "accept", "src": ["group:devops"], "dst": ["tag:server"], "users": ["*"]}
  ],

  "tests": [
    {"src": "group:engineering", "accept": ["tag:dev:22"]},
    {"src": "group:engineering", "deny": ["tag:prod:22"]}
  ]
}
```

## Best Practices

1. **Start restrictive** - Begin with deny-all, add rules as needed
2. **Use groups** - Easier to manage than individual users
3. **Use tags for servers** - Decouple from user accounts
4. **Write tests** - Prevent accidental lockouts
5. **Review regularly** - Audit access quarterly
6. **Document changes** - Use comments in HuJSON

## Editing ACLs

- **Admin Console**: https://login.tailscale.com/admin/acls
- **GitOps**: Version control your policies
- **API**: Programmatic updates

```bash
# Validate before applying
tailscale debug policy-file /path/to/policy.json
```
