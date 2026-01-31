# Portainer Container Management

Web UI for managing Docker containers.

## When to Use

- Visual container management
- Deploy stacks from templates
- Manage multiple Docker hosts
- Less CLI, more GUI

## Installation

### Docker Compose

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - portainer_data:/data
    ports:
      - 9443:9443  # HTTPS UI
      - 9000:9000  # HTTP UI (optional)
      - 8000:8000  # Edge Agent

volumes:
  portainer_data:
```

### First Run

1. Navigate to `https://localhost:9443`
2. Create admin user
3. Choose "Local" environment

## Managing Containers

### Deploy from UI

Containers → Add Container:
- Image: `nginx:latest`
- Port mapping: 8080:80
- Volumes: Mount paths
- Environment variables

### Deploy Stack (Compose)

Stacks → Add Stack:
```yaml
services:
  whoami:
    image: traefik/whoami
    ports:
      - "8080:80"
```

### App Templates

Settings → App Templates:
- Add custom template URL
- Community templates available

## Stack Management

### From Git Repository

Stacks → Add Stack → Git Repository:
- Repository URL
- Compose path: `docker-compose.yml`
- Auto-update: Optional

### Environment Variables

Use `.env` format in stack definition:
```yaml
services:
  app:
    image: myapp:${VERSION:-latest}
    environment:
      - API_KEY=${API_KEY}
```

Define in Portainer: Stack → Environment variables

## Multi-Host Management

### Add Remote Docker

Environments → Add Environment → Docker Standalone:
- Via Edge Agent (recommended for remote)
- Via API (direct socket access)

### Edge Agent Setup

On remote host:
```yaml
services:
  edge-agent:
    image: portainer/agent:latest
    environment:
      - EDGE=1
      - EDGE_ID=<unique-id>
      - EDGE_KEY=<key-from-portainer>
      - EDGE_INSECURE_POLL=1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    restart: always
```

## Custom Templates

### Template Definition

```json
{
  "version": "2",
  "templates": [
    {
      "type": 1,
      "title": "Nginx",
      "description": "Web server",
      "logo": "https://example.com/nginx.png",
      "image": "nginx:latest",
      "ports": ["80/tcp"],
      "volumes": [
        {
          "container": "/usr/share/nginx/html",
          "bind": "/data/nginx"
        }
      ],
      "env": [
        {
          "name": "NGINX_HOST",
          "label": "Hostname"
        }
      ]
    }
  ]
}
```

### Stack Template

```json
{
  "type": 2,
  "title": "WordPress",
  "description": "WordPress with MySQL",
  "repository": {
    "url": "https://github.com/portainer/templates",
    "stackfile": "stacks/wordpress/docker-compose.yml"
  }
}
```

## Backup & Restore

### Backup Data

```bash
# Stop Portainer
docker stop portainer

# Backup volume
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer-backup.tar.gz /data

# Start Portainer
docker start portainer
```

### Restore

```bash
docker stop portainer
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/portainer-backup.tar.gz -C /"
docker start portainer
```

## Access Control

### Teams & Users

Settings → Users:
- Create team (e.g., "Developers")
- Add users to team
- Assign environment access

### Role-Based Access

| Role | Permissions |
|------|-------------|
| Admin | Full access |
| Standard | Manage own resources |
| Restricted | View only |
| Helpdesk | Limited management |

### Resource Access

Environments → [Env] → Access Management:
- Assign teams to environments
- Set default role

## API Access

### Generate Token

Settings → Users → [User] → Access tokens

### API Examples

```bash
# List containers
curl -X GET "https://portainer:9443/api/endpoints/1/docker/containers/json" \
  -H "X-API-Key: your-token"

# Deploy stack
curl -X POST "https://portainer:9443/api/stacks" \
  -H "X-API-Key: your-token" \
  -H "Content-Type: application/json" \
  -d '{"Name":"mystack","StackFileContent":"services:\n  web:\n    image: nginx"}'
```

## Integrations

### Webhooks

Containers → [Container] → Webhooks:
- Create webhook
- Trigger: `POST https://portainer:9443/api/webhooks/<id>`

Use for CI/CD:
```bash
# Redeploy on push
curl -X POST https://portainer:9443/api/webhooks/abc123
```

### Registry Management

Registries → Add Registry:
- Docker Hub (credentials)
- Private registry
- ECR, ACR, GCR

## Monitoring

### Built-in Stats

Containers → [Container] → Stats:
- CPU usage
- Memory usage
- Network I/O
- Disk I/O

### Logs

Containers → [Container] → Logs:
- Real-time log streaming
- Download logs
- Search/filter

## Troubleshooting

### Can't Connect to Docker

```bash
# Check socket permissions
ls -la /var/run/docker.sock

# Add user to docker group
sudo usermod -aG docker $USER
```

### Edge Agent Not Connecting

1. Check EDGE_KEY matches Portainer
2. Verify network connectivity
3. Check agent logs: `docker logs edge-agent`

### Stack Deploy Fails

1. Validate compose syntax
2. Check image availability
3. Review deployment logs in stack details

## Best Practices

1. **Use stacks** - version control your compose files
2. **Set resource limits** - prevent runaway containers
3. **Enable auto-update** - keep images current (with caution)
4. **Backup regularly** - Portainer data volume
5. **Use HTTPS** - expose only 9443
6. **Restrict access** - role-based permissions
7. **Template everything** - consistent deployments
