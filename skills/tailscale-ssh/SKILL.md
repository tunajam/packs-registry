# tailscale-ssh

Tailscale SSH - keyless SSH authentication and connection management

## Source

Tailscale SSH documentation: https://tailscale.com/kb/1193/tailscale-ssh

## Overview

Tailscale SSH replaces traditional SSH key management with Tailscale's identity system:
- No more SSH key distribution
- Access controlled by tailnet ACLs
- Optional session recording
- Check mode for sensitive access

## How It Works

1. Tailscale intercepts SSH connections on port 22 (Tailscale IP only)
2. Uses WireGuard for encryption
3. Authenticates via Tailscale identity
4. Authorizes via SSH ACL rules
5. Traditional SSH (non-Tailscale IPs) continues to work

## Enable SSH Server

```bash
# On the machine you want to SSH into
tailscale up --ssh

# Or with tailscale set
tailscale set --ssh
```

This generates SSH host keys and enables Tailscale SSH.

## Connect via SSH

```bash
# Using MagicDNS hostname
ssh user@my-server

# Using Tailscale IP
ssh user@100.x.y.z

# Tailscale manages authentication - no keys needed
```

## SSH ACL Policies

Configure SSH access in your tailnet policy file:

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:server"],
      "users": ["root", "deploy"]
    }
  ]
}
```

### Policy Fields

| Field | Description |
|-------|-------------|
| `action` | `accept` or `check` |
| `src` | Who can connect (users, groups, tags) |
| `dst` | Target machines (tags, autogroup:self) |
| `users` | Allowed Unix usernames |
| `checkPeriod` | Re-auth interval for check mode |

### Actions

**accept**: Allow connection with current Tailscale auth
```json
{"action": "accept", "src": ["group:devops"], "dst": ["tag:server"], "users": ["*"]}
```

**check**: Require re-authentication periodically
```json
{
  "action": "check",
  "checkPeriod": "12h",
  "src": ["group:devops"],
  "dst": ["tag:prod"],
  "users": ["root"]
}
```

### User Selectors

| Selector | Meaning |
|----------|---------|
| `*` | Any user including root |
| `autogroup:nonroot` | Any user except root |
| `localpart:*@example.com` | Map email local part to Unix user |
| `["ubuntu", "deploy"]` | Specific users |

## Common SSH ACL Patterns

### Users Access Own Devices
```json
{
  "ssh": [
    {
      "action": "check",
      "src": ["autogroup:member"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

### DevOps Full Access to Servers
```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:server"],
      "users": ["*"]
    }
  ]
}
```

### Engineers Access Dev (Non-Root)
```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:dev"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

### Prod Access with Re-Auth
```json
{
  "ssh": [
    {
      "action": "check",
      "checkPeriod": "4h",
      "src": ["group:devops"],
      "dst": ["tag:prod"],
      "users": ["*"]
    }
  ]
}
```

### Email-to-Username Mapping
```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["user:*@example.com"],
      "dst": ["tag:server"],
      "users": ["localpart:*@example.com"]
    }
  ]
}
```
`alice@example.com` → logs in as `alice`.

## Session Recording

Record SSH sessions for audit/compliance:

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:prod"],
      "users": ["*"],
      "recorder": ["tag:recorder"]
    }
  ]
}
```

Set up a recording server:
```bash
# On recorder machine
tailscale up --advertise-tags=tag:recorder
# Configure recording storage
```

## Check Mode Details

Check mode requires re-authentication via identity provider:

```json
{
  "action": "check",
  "checkPeriod": "12h",  // Default
  "src": ["group:security"],
  "dst": ["tag:sensitive"],
  "users": ["root"]
}
```

### Check Periods
- Minimum: `1m` (one minute)
- Maximum: `168h` (one week)
- Default: `12h`
- `always` - check every connection (may break automation)

## Environment Variables

Allow clients to send environment variables:

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:server"],
      "users": ["*"],
      "acceptEnv": ["LC_*", "LANG", "EDITOR"]
    }
  ]
}
```

Wildcards: `*` matches any chars, `?` matches single char.

## Disable SSH

```bash
# On the server
tailscale set --ssh=false
```

## Combined ACL + SSH Example

```json
{
  "groups": {
    "group:devops": ["alice@example.com", "bob@example.com"],
    "group:engineering": ["carol@example.com"]
  },

  "tagOwners": {
    "tag:server": ["group:devops"],
    "tag:dev": ["group:devops"],
    "tag:prod": ["group:devops"]
  },

  "acls": [
    // Network access to servers
    {"action": "accept", "src": ["group:devops"], "dst": ["tag:server:*"]},
    {"action": "accept", "src": ["group:engineering"], "dst": ["tag:dev:*"]}
  ],

  "ssh": [
    // DevOps: full SSH to all servers
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["tag:server"],
      "users": ["*"]
    },
    // Engineering: non-root SSH to dev only
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:dev"],
      "users": ["autogroup:nonroot"]
    },
    // Prod requires check mode
    {
      "action": "check",
      "checkPeriod": "4h",
      "src": ["group:devops"],
      "dst": ["tag:prod"],
      "users": ["root"]
    }
  ]
}
```

## Key Rotation

Rotate SSH host keys:

```bash
# Re-authenticate generates new keys
tailscale logout
tailscale up --ssh
```

## Limitations

- Server: Linux, macOS (open-source variant only)
- Client: Any platform
- Only port 22
- No password auth (Tailscale identity only)
- `autogroup:self` - users can't SSH to others' devices

## Troubleshooting

### Can't Connect
```bash
# Check SSH is enabled
tailscale status --json | jq '.Self.CapMap["ssh"]'

# Verify ACLs allow SSH
# Check both network ACL and SSH ACL
```

### Permission Denied
- Verify Unix user exists on target
- Check SSH ACL allows that user
- Verify src/dst match your identity/target

### Check Mode Not Working
- Ensure identity provider session active
- Try logging out/in to Tailscale
- Verify checkPeriod hasn't passed

## vs Traditional SSH

| Feature | Traditional SSH | Tailscale SSH |
|---------|-----------------|---------------|
| Authentication | Keys/passwords | Tailscale identity |
| Key distribution | Manual | Automatic |
| Access control | authorized_keys | Centralized ACLs |
| Encryption | SSH | WireGuard + SSH |
| User revocation | Remove keys | Update ACL |
| Audit | sshd logs | Session recording |

## Best Practices

1. **Use check mode for prod** - Require re-auth for sensitive systems
2. **Restrict root** - Use `autogroup:nonroot` by default
3. **Enable session recording** - For compliance and audit
4. **Tag your servers** - Manage access by role, not machine
5. **Test ACL changes** - Use policy tests before applying
6. **Keep traditional SSH** - As backup on non-Tailscale IP
