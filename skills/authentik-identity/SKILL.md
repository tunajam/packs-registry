# Authentik Identity Provider

Full-featured identity provider for homelab SSO.

## When to Use

- Need OIDC/SAML provider
- Want LDAP for legacy apps
- Single sign-on across services
- User self-service portal

## Authentik vs Authelia

| Feature | Authentik | Authelia |
|---------|-----------|----------|
| Forward Auth | Yes | Yes |
| OIDC Provider | Yes | No |
| SAML Provider | Yes | No |
| LDAP Outpost | Yes | No |
| User Portal | Yes | Basic |
| Complexity | Higher | Lower |

**Choose Authentik for**: Apps that need OIDC (Grafana, Nextcloud native)
**Choose Authelia for**: Simple forward auth protection

## Docker Setup

```yaml
services:
  postgresql:
    image: postgres:16-alpine
    container_name: authentik-db
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - ./database:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=authentik
      - POSTGRES_PASSWORD=changeme
      - POSTGRES_DB=authentik
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: authentik-redis
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-server
    command: server
    environment:
      - AUTHENTIK_REDIS__HOST=redis
      - AUTHENTIK_POSTGRESQL__HOST=postgresql
      - AUTHENTIK_POSTGRESQL__USER=authentik
      - AUTHENTIK_POSTGRESQL__NAME=authentik
      - AUTHENTIK_POSTGRESQL__PASSWORD=changeme
      - AUTHENTIK_SECRET_KEY=changeme-generate-random-string
    volumes:
      - ./media:/media
      - ./templates:/templates
    ports:
      - "9000:9000"
      - "9443:9443"
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    command: worker
    environment:
      - AUTHENTIK_REDIS__HOST=redis
      - AUTHENTIK_POSTGRESQL__HOST=postgresql
      - AUTHENTIK_POSTGRESQL__USER=authentik
      - AUTHENTIK_POSTGRESQL__NAME=authentik
      - AUTHENTIK_POSTGRESQL__PASSWORD=changeme
      - AUTHENTIK_SECRET_KEY=changeme-generate-random-string
    volumes:
      - ./media:/media
      - ./templates:/templates
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
```

### Generate Secret Key

```bash
openssl rand -base64 36
```

## Initial Setup

1. Navigate to `http://your-ip:9000/if/flow/initial-setup/`
2. Create admin account
3. Login at `http://your-ip:9000/if/admin/`

## Applications

### Create Application

Applications → Create:
- Name: Grafana
- Slug: grafana
- Provider: Create new

### Application Flows

- Authorization flow: default-provider-authorization-implicit
- Invalidation flow: default-provider-invalidation

## Providers

### OAuth2/OIDC Provider

Providers → Create → OAuth2/OpenID Provider:
- Name: Grafana OIDC
- Authentication flow: default-authentication-flow
- Authorization flow: default-provider-authorization-implicit
- Client ID: auto-generated
- Redirect URIs: `https://grafana.example.com/login/generic_oauth`

### Forward Auth (Proxy Provider)

Providers → Create → Proxy Provider:
- Name: Whoami Proxy
- Authentication flow: default-authentication-flow
- Mode: Forward auth (domain level)
- External host: `https://whoami.example.com`
- Cookie domain: `example.com`

## Outposts

### Embedded Outpost

Included by default for forward auth.

### LDAP Outpost

Outposts → Create:
- Name: LDAP
- Type: LDAP
- Integration: Embedded

Configure:
- Applications → Add LDAP application
- Bind DN: `cn=ldap,ou=users,dc=ldap,dc=example,dc=com`

## Traefik Integration

### Forward Auth Middleware

```yaml
# traefik dynamic config
http:
  middlewares:
    authentik:
      forwardAuth:
        address: http://authentik-server:9000/outpost.goauthentik.io/auth/traefik
        trustForwardHeader: true
        authResponseHeaders:
          - X-authentik-username
          - X-authentik-groups
          - X-authentik-email
          - X-authentik-name
          - X-authentik-uid
```

### Protected Service

```yaml
services:
  whoami:
    labels:
      - "traefik.http.routers.whoami.middlewares=authentik@file"
```

## Grafana OIDC

### Authentik Setup

1. Create OAuth2 provider for Grafana
2. Note Client ID and Client Secret
3. Add redirect URI: `https://grafana.example.com/login/generic_oauth`

### Grafana Config

```ini
[auth.generic_oauth]
enabled = true
name = Authentik
client_id = <client-id>
client_secret = <client-secret>
scopes = openid email profile
auth_url = https://auth.example.com/application/o/authorize/
token_url = https://auth.example.com/application/o/token/
api_url = https://auth.example.com/application/o/userinfo/
role_attribute_path = contains(groups[*], 'Grafana Admins') && 'Admin' || 'Viewer'
```

## Nextcloud OIDC

### Authentik Provider

Create OAuth2 provider with scopes: openid, email, profile

### Nextcloud Setup

1. Install "Social Login" or "OpenID Connect" app
2. Configure OIDC:
   - Client ID: from Authentik
   - Client Secret: from Authentik
   - Authorize URL: `https://auth.example.com/application/o/authorize/`
   - Token URL: `https://auth.example.com/application/o/token/`
   - Userinfo URL: `https://auth.example.com/application/o/userinfo/`

## LDAP Integration

### For Legacy Apps

1. Create LDAP Provider
2. Create LDAP Outpost
3. Configure app with:
   - LDAP Host: `auth.example.com`
   - LDAP Port: 636 (or 389)
   - Base DN: `dc=ldap,dc=example,dc=com`
   - Bind DN: `cn=admin,ou=users,dc=ldap,dc=example,dc=com`

## User Management

### Create User

Directory → Users → Create:
- Username
- Name
- Email
- Groups

### Groups

Directory → Groups → Create:
- Name: admins
- Members: add users

### Invitations

Flows → Enrollment flows:
- Create invitation-based enrollment
- Send invite link to new users

## Multi-Factor Authentication

### Enable MFA

Flows → Stages:
- Create authenticator_totp stage
- Add to authentication flow

### Per-User MFA

Users can self-enroll at:
`https://auth.example.com/if/user/`

## Troubleshooting

### Check Logs

```bash
docker logs authentik-server
docker logs authentik-worker
```

### OIDC Debugging

Test OIDC flow:
`https://auth.example.com/application/o/<app-slug>/.well-known/openid-configuration`

### Common Issues

**Redirect loop:**
- Check cookie domain matches
- Verify external host URL

**LDAP bind fails:**
- Check outpost is running
- Verify bind DN format

**OIDC token invalid:**
- Check client secret
- Verify redirect URI

## Best Practices

1. **Use OIDC when possible** - native integration is better
2. **Forward auth for simple protection** - no app changes needed
3. **Organize with groups** - easier permission management
4. **Enable MFA** - especially for admin accounts
5. **Regular backups** - PostgreSQL database
6. **Separate by environment** - dev/prod applications
7. **Monitor outposts** - ensure they're healthy
