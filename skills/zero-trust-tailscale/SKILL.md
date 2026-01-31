# zero-trust-tailscale

Zero Trust networking patterns and best practices with Tailscale

## Source

Tailscale Zero Trust: https://tailscale.com/kb/1123/zero-trust

## Zero Trust Principles

1. **Never trust, always verify** - No implicit trust based on network location
2. **Least privilege access** - Minimum access needed for the task
3. **Assume breach** - Design as if attackers are already inside
4. **Verify explicitly** - Authenticate and authorize every request

## Tailscale's Zero Trust Model

### Identity-Based Access
- Every connection tied to authenticated identity
- No anonymous access possible
- Access controlled by WHO, not WHERE

### Device Authentication
- Devices authenticated via identity provider
- Node keys verify device identity
- Compromised device can be revoked instantly

### Deny by Default
- No traffic allowed without explicit ACL rule
- Network segmentation built-in
- Principle of least privilege enforced

### End-to-End Encryption
- WireGuard encryption for all traffic
- No central decryption point
- No traffic inspection by intermediaries

## Implementation Patterns

### 1. Remove Default Allow

Start with empty ACLs:
```json
{
  "acls": []
}
```

Then add only what's needed:
```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:dev:22,443"]
    }
  ]
}
```

### 2. Role-Based Access Control

Define groups by job function:
```json
{
  "groups": {
    "group:engineering": ["alice@example.com", "bob@example.com"],
    "group:devops": ["carol@example.com"],
    "group:security": ["dan@example.com"]
  },

  "acls": [
    {"action": "accept", "src": ["group:engineering"], "dst": ["tag:dev:*"]},
    {"action": "accept", "src": ["group:devops"], "dst": ["tag:dev:*", "tag:staging:*", "tag:prod:*"]},
    {"action": "accept", "src": ["group:security"], "dst": ["*:*"]}
  ]
}
```

### 3. Resource Tagging

Tag by sensitivity/environment:
```json
{
  "tagOwners": {
    "tag:public": ["group:devops"],
    "tag:internal": ["group:devops"],
    "tag:confidential": ["group:security"],
    "tag:dev": ["group:devops"],
    "tag:staging": ["group:devops"],
    "tag:prod": ["group:devops", "group:security"]
  }
}
```

### 4. Micro-Segmentation

Isolate services that don't need to communicate:
```json
{
  "acls": [
    // Web tier can reach API tier
    {"action": "accept", "src": ["tag:web"], "dst": ["tag:api:8080"]},
    
    // API tier can reach database
    {"action": "accept", "src": ["tag:api"], "dst": ["tag:database:5432"]},
    
    // No direct web-to-database access
    // (implicitly denied)
  ]
}
```

### 5. SSH with Check Mode

Require re-authentication for sensitive access:
```json
{
  "ssh": [
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

### 6. Time-Limited Access

Use auth key expiry for temporary access:
```bash
# Generate short-lived key
# Access automatically revoked when key expires
```

### 7. Device Posture

Restrict access based on device state:
```json
{
  "postures": {
    "posture:corporate": ["node:os == 'macos'", "node:tsVersion >= '1.40'"]
  },

  "acls": [
    {
      "action": "accept",
      "src": ["group:employees"],
      "dst": ["tag:internal:*"],
      "srcPosture": ["posture:corporate"]
    }
  ]
}
```

## Common Zero Trust Scenarios

### Remote Worker Access

```json
{
  "groups": {
    "group:employees": ["*@example.com"]
  },

  "acls": [
    // Employees access internal tools
    {"action": "accept", "src": ["group:employees"], "dst": ["tag:internal:443"]},
    
    // No access to production without DevOps role
    // (production access explicitly denied by default)
  ],

  "ssh": [
    // Check mode for any server SSH
    {
      "action": "check",
      "src": ["group:employees"],
      "dst": ["tag:server"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

### Contractor Access

```json
{
  "groups": {
    "group:contractors": ["contractor1@external.com"]
  },

  "acls": [
    // Contractors only access specific dev resources
    {"action": "accept", "src": ["group:contractors"], "dst": ["tag:contractor-accessible:*"]}
  ]
}
```

### Service-to-Service

```json
{
  "tagOwners": {
    "tag:payment-service": ["group:devops"],
    "tag:user-service": ["group:devops"],
    "tag:database": ["group:devops"]
  },

  "acls": [
    // Payment service can only reach database
    {"action": "accept", "src": ["tag:payment-service"], "dst": ["tag:database:5432"]},
    
    // User service can reach database
    {"action": "accept", "src": ["tag:user-service"], "dst": ["tag:database:5432"]},
    
    // Services cannot reach each other
    // (no lateral movement)
  ]
}
```

### Multi-Cloud

```json
{
  "acls": [
    // AWS services can reach GCP database
    {"action": "accept", "src": ["tag:aws-service"], "dst": ["tag:gcp-database:5432"]},
    
    // GCP services can reach AWS queue
    {"action": "accept", "src": ["tag:gcp-service"], "dst": ["tag:aws-queue:5672"]}
  ]
}
```

## Security Hardening

### 1. Use Tags, Not Users

```json
// ❌ Avoid
{"action": "accept", "src": ["alice@example.com"], "dst": ["100.x.y.z:*"]}

// ✅ Prefer
{"action": "accept", "src": ["group:admins"], "dst": ["tag:admin-servers:*"]}
```

### 2. Principle of Least Privilege

```json
// ❌ Too permissive
{"action": "accept", "src": ["*"], "dst": ["*:*"]}

// ✅ Minimal access
{"action": "accept", "src": ["group:web-team"], "dst": ["tag:web:80,443"]}
```

### 3. Port Restrictions

```json
// ❌ All ports
{"action": "accept", "src": ["group:devops"], "dst": ["tag:server:*"]}

// ✅ Specific ports
{"action": "accept", "src": ["group:devops"], "dst": ["tag:server:22,80,443"]}
```

### 4. Write ACL Tests

```json
{
  "tests": [
    // Verify contractors can't reach prod
    {"src": "group:contractors", "deny": ["tag:prod:*"]},
    
    // Verify engineers can reach dev
    {"src": "group:engineering", "accept": ["tag:dev:22"]}
  ]
}
```

### 5. Regular Access Reviews

- Audit groups quarterly
- Remove departed employees immediately
- Review tag assignments
- Check for overly permissive rules

### 6. Enable Audit Logging

- Network flow logs
- Configuration change logs
- SSH session recording for sensitive access

## Migration Strategy

### Phase 1: Inventory
1. Document current network access
2. Identify all resources and their sensitivity
3. Map users to required access

### Phase 2: Design
1. Define groups based on job functions
2. Create tag taxonomy for resources
3. Write ACL policy (restrictive)
4. Write tests to validate

### Phase 3: Implement
1. Deploy Tailscale on resources
2. Apply tags to resources
3. Add users to groups
4. Enable ACLs (initially permissive)

### Phase 4: Harden
1. Gradually restrict ACLs
2. Monitor for access issues
3. Remove unused rules
4. Enable check mode for sensitive access

## Monitoring & Response

### Detect Anomalies
- Monitor network flow logs
- Alert on unusual access patterns
- Track failed connection attempts

### Incident Response
```bash
# Immediately revoke device
# In Admin Console: Machines → Device → Remove

# Or expire all user sessions
# Admin Console → Users → User → Expire keys
```

### Audit Trail
- All policy changes logged
- Device authentication logged
- SSH sessions optionally recorded

## Best Practices Summary

1. **Start restrictive** - Add access as needed, not remove later
2. **Use groups** - Easier to manage than individuals
3. **Tag everything** - Resources should be tagged by role/sensitivity
4. **Test your ACLs** - Write tests to prevent regressions
5. **Check mode for sensitive** - Re-auth for prod/root access
6. **Regular audits** - Review access quarterly
7. **Log everything** - Enable flow logs and session recording
8. **Automate onboarding** - Use identity provider groups
9. **Plan for incidents** - Know how to revoke quickly
10. **Document decisions** - Comments in ACL file explaining rules
