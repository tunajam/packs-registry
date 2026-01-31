# Authelia SSO

Single sign-on and two-factor authentication for homelab.

## When to Use

- Protect services that lack built-in auth
- Centralized login for all homelab apps
- Add 2FA to existing services
- Replace HTTP basic auth

## Architecture

```
Internet → Reverse Proxy (Traefik/Caddy) → Authelia → Protected Service
```

Authelia intercepts requests and verifies authentication before forwarding to the protected service.

## Docker Setup

```yaml
services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    volumes:
      - ./config:/config
    environment:
      - TZ=America/Los_Angeles
    ports:
      - "9091:9091"
    restart: unless-stopped
    depends_on:
      - redis

  redis:
    image: redis:alpine
    container_name: authelia-redis
    volumes:
      - ./redis:/data
    restart: unless-stopped
```

## Configuration

### config/configuration.yml

```yaml
---
server:
  address: 'tcp://0.0.0.0:9091'

log:
  level: info

totp:
  issuer: 'homelab.example.com'

authentication_backend:
  file:
    path: '/config/users_database.yml'
    password:
      algorithm: argon2id
      iterations: 1
      memory: 1024
      parallelism: 8
      key_length: 32
      salt_length: 16

access_control:
  default_policy: deny
  rules:
    # Public endpoints
    - domain: 'public.example.com'
      policy: bypass
    
    # Admin-only services
    - domain: 'admin.example.com'
      policy: two_factor
      subject:
        - 'group:admins'
    
    # One-factor for most services
    - domain: '*.example.com'
      policy: one_factor

session:
  name: authelia_session
  secret: 'insecure_session_secret'  # Change this!
  expiration: 3600
  inactivity: 300
  domain: 'example.com'
  redis:
    host: 'redis'
    port: 6379

regulation:
  max_retries: 3
  find_time: 120
  ban_time: 300

storage:
  local:
    path: '/config/db.sqlite3'

notifier:
  filesystem:
    filename: '/config/notification.txt'
  # Or use SMTP:
  # smtp:
  #   host: 'smtp.example.com'
  #   port: 587
  #   username: 'authelia@example.com'
  #   password: 'password'
  #   sender: 'Authelia <authelia@example.com>'
```

### config/users_database.yml

```yaml
users:
  admin:
    displayname: "Admin User"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."
    email: admin@example.com
    groups:
      - admins
      - users
  
  john:
    displayname: "John Doe"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."
    email: john@example.com
    groups:
      - users
```

### Generate Password Hash

```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 --password 'yourpassword'
```

## Traefik Integration

### docker-compose.yml (Authelia labels)

```yaml
services:
  authelia:
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.authelia.rule=Host(`auth.example.com`)'
      - 'traefik.http.routers.authelia.entrypoints=websecure'
      - 'traefik.http.routers.authelia.tls.certresolver=letsencrypt'
      - 'traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.example.com'
      - 'traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true'
      - 'traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email'
```

### Protected Service

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.whoami.rule=Host(`whoami.example.com`)'
      - 'traefik.http.routers.whoami.entrypoints=websecure'
      - 'traefik.http.routers.whoami.tls.certresolver=letsencrypt'
      - 'traefik.http.routers.whoami.middlewares=authelia@docker'
```

## Caddy Integration

### Caddyfile

```caddyfile
(authelia) {
    forward_auth authelia:9091 {
        uri /api/verify?rd=https://auth.example.com
        copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
    }
}

auth.example.com {
    reverse_proxy authelia:9091
}

protected.example.com {
    import authelia
    reverse_proxy backend:8080
}
```

## Access Policies

### Policy Levels

| Policy | Description |
|--------|-------------|
| `bypass` | No authentication |
| `one_factor` | Password only |
| `two_factor` | Password + TOTP/WebAuthn |
| `deny` | Block access |

### Complex Rules

```yaml
access_control:
  rules:
    # API endpoints - bypass for automation
    - domain: 'app.example.com'
      resources:
        - '^/api/.*'
      policy: bypass
    
    # Admin panel - 2FA required
    - domain: 'app.example.com'
      resources:
        - '^/admin.*'
      policy: two_factor
    
    # Main app - 1FA
    - domain: 'app.example.com'
      policy: one_factor
```

### Network-Based Rules

```yaml
access_control:
  rules:
    # Local network bypass
    - domain: 'internal.example.com'
      networks:
        - '192.168.1.0/24'
        - '10.0.0.0/8'
      policy: bypass
    
    # External requires 2FA
    - domain: 'internal.example.com'
      policy: two_factor
```

## Two-Factor Setup

### TOTP (Authenticator App)

1. Login to Authelia portal
2. Click "Register device"
3. Scan QR code with authenticator app
4. Enter 6-digit code to verify

### WebAuthn (Hardware Keys)

```yaml
webauthn:
  disable: false
  display_name: 'Homelab'
  attestation_conveyance_preference: 'indirect'
  user_verification: 'preferred'
```

## LDAP Backend (Optional)

```yaml
authentication_backend:
  ldap:
    implementation: custom
    url: 'ldap://openldap:389'
    base_dn: 'dc=example,dc=com'
    username_attribute: 'uid'
    additional_users_dn: 'ou=users'
    users_filter: '(&({username_attribute}={input})(objectClass=person))'
    additional_groups_dn: 'ou=groups'
    groups_filter: '(&(member={dn})(objectClass=groupOfNames))'
    group_name_attribute: 'cn'
    mail_attribute: 'mail'
    display_name_attribute: 'displayName'
    user: 'cn=admin,dc=example,dc=com'
    password: 'password'
```

## Password Reset

Enable SMTP notifier:

```yaml
notifier:
  smtp:
    host: 'smtp.example.com'
    port: 587
    username: 'authelia@example.com'
    password: 'password'
    sender: 'Authelia <authelia@example.com>'
```

Users can reset via: `https://auth.example.com/reset-password/step1`

## Troubleshooting

### Check Logs

```bash
docker logs -f authelia
```

### Test Authentication

```bash
# Test forward auth
curl -I -H "X-Forwarded-For: 1.2.3.4" \
  "http://localhost:9091/api/verify?rd=https://auth.example.com"
```

### Common Issues

**"Access Denied" for all:**
- Check `default_policy` in access_control
- Verify domain matches exactly

**Session not persisting:**
- Check session domain matches cookie domain
- Ensure Redis is running

**2FA not working:**
- Verify TOTP secret is registered
- Check system time sync (NTP)

## Best Practices

1. **Use Redis** - required for HA and better performance
2. **Strong secrets** - generate random session/encryption secrets
3. **Start with one_factor** - add 2FA gradually
4. **Backup config** - users_database.yml and db.sqlite3
5. **Use SMTP** - enable password reset
6. **Test bypass rules** - ensure they don't expose sensitive services
7. **Monitor failed logins** - set up alerting
