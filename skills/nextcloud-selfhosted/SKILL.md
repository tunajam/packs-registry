# Nextcloud Self-Hosted Cloud

Your own cloud for files, calendar, contacts, and more.

## When to Use

- Replace Google Drive/Dropbox
- Self-host calendar and contacts
- Collaborative document editing
- Photo backup from mobile

## Docker Setup

### Full Stack (Recommended)

```yaml
services:
  nextcloud:
    image: nextcloud:apache
    container_name: nextcloud
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=changeme
      - REDIS_HOST=redis
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=changeme
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.example.com
      - OVERWRITEPROTOCOL=https
      - OVERWRITEHOST=cloud.example.com
    volumes:
      - ./nextcloud:/var/www/html
      - ./data:/var/www/html/data
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    container_name: nextcloud-db
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=changeme
    volumes:
      - ./db:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  cron:
    image: nextcloud:apache
    container_name: nextcloud-cron
    volumes:
      - ./nextcloud:/var/www/html
      - ./data:/var/www/html/data
    entrypoint: /cron.sh
    depends_on:
      - nextcloud
    restart: unless-stopped
```

### LinuxServer.io Image

```yaml
services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
      - ./data:/data
    ports:
      - 443:443
    restart: unless-stopped
```

## Configuration

### config.php Tweaks

Add to `nextcloud/config/config.php`:

```php
<?php
$CONFIG = array (
  // ... existing config ...
  
  // Performance
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => [
    'host' => 'redis',
    'port' => 6379,
  ],
  
  // File handling
  'default_phone_region' => 'US',
  'trashbin_retention_obligation' => 'auto, 30',
  
  // Security
  'overwriteprotocol' => 'https',
  
  // Preview generation
  'preview_max_x' => 1024,
  'preview_max_y' => 1024,
  'enable_previews' => true,
  'enabledPreviewProviders' => [
    'OC\\Preview\\PNG',
    'OC\\Preview\\JPEG',
    'OC\\Preview\\GIF',
    'OC\\Preview\\HEIC',
    'OC\\Preview\\Movie',
    'OC\\Preview\\MP4',
    'OC\\Preview\\MKV',
  ],
);
```

### Reverse Proxy Headers

For Traefik/Caddy:

```php
'trusted_proxies' => ['172.16.0.0/12', '192.168.0.0/16'],
'overwritehost' => 'cloud.example.com',
'overwriteprotocol' => 'https',
'forwarded_for_headers' => ['HTTP_X_FORWARDED_FOR'],
```

## Essential Apps

### Install via CLI

```bash
docker exec -u www-data nextcloud php occ app:install calendar
docker exec -u www-data nextcloud php occ app:install contacts
docker exec -u www-data nextcloud php occ app:install tasks
docker exec -u www-data nextcloud php occ app:install notes
docker exec -u www-data nextcloud php occ app:install deck
docker exec -u www-data nextcloud php occ app:install photos
docker exec -u www-data nextcloud php occ app:install memories
```

### Recommended Apps

| App | Purpose |
|-----|---------|
| Calendar | CalDAV calendar sync |
| Contacts | CardDAV contacts sync |
| Tasks | To-do lists |
| Notes | Markdown notes |
| Deck | Kanban boards |
| Photos | Photo management |
| Memories | Photo timeline with AI |
| Talk | Video calls, chat |
| Office | Document editing |
| Music | Music streaming |

## Background Jobs

### Configure Cron

```bash
# Check cron status
docker exec -u www-data nextcloud php occ background:cron

# Run cron manually
docker exec -u www-data nextcloud php -f cron.php
```

System cron (alternative to cron container):
```cron
*/5 * * * * docker exec -u www-data nextcloud php -f cron.php
```

## External Storage

### S3/MinIO

Settings → External Storage:
- Folder name: archive
- External storage: Amazon S3
- Bucket: nextcloud-backup
- Region: us-east-1
- Access key: your-key
- Secret key: your-secret

### Local NAS

```bash
docker exec -u www-data nextcloud php occ files_external:create \
  nas local null::null \
  -c datadir=/mnt/nas
```

## Performance Optimization

### PHP Memory Limit

Add to docker-compose:
```yaml
environment:
  - PHP_MEMORY_LIMIT=1G
  - PHP_UPLOAD_LIMIT=10G
```

### Enable APCu

In `config.php`:
```php
'memcache.local' => '\\OC\\Memcache\\APCu',
```

### Preview Generation

Pre-generate previews:
```bash
docker exec -u www-data nextcloud php occ preview:generate-all -vvv
```

### Database Maintenance

```bash
# Add missing indices
docker exec -u www-data nextcloud php occ db:add-missing-indices

# Convert to big int
docker exec -u www-data nextcloud php occ db:convert-filecache-bigint

# Cleanup
docker exec -u www-data nextcloud php occ files:cleanup
docker exec -u www-data nextcloud php occ trashbin:cleanup --all-users
```

## Mobile Sync

### Files

Install Nextcloud app → Add account → Enter server URL

### Calendar/Contacts (iOS)

Settings → Calendar → Add Account → Other:
- CalDAV: `https://cloud.example.com/remote.php/dav`
- CardDAV: `https://cloud.example.com/remote.php/dav`

### Calendar/Contacts (Android)

Use DAVx⁵ app:
- Base URL: `https://cloud.example.com`
- Login with Nextcloud credentials

## CLI Commands (occ)

```bash
# List all commands
docker exec -u www-data nextcloud php occ list

# User management
docker exec -u www-data nextcloud php occ user:list
docker exec -u www-data nextcloud php occ user:add username
docker exec -u www-data nextcloud php occ user:resetpassword username

# File operations
docker exec -u www-data nextcloud php occ files:scan --all
docker exec -u www-data nextcloud php occ files:cleanup

# Maintenance
docker exec -u www-data nextcloud php occ maintenance:mode --on
docker exec -u www-data nextcloud php occ maintenance:mode --off
docker exec -u www-data nextcloud php occ upgrade

# App management
docker exec -u www-data nextcloud php occ app:list
docker exec -u www-data nextcloud php occ app:update --all
```

## Backup & Restore

### Backup

```bash
# Maintenance mode
docker exec -u www-data nextcloud php occ maintenance:mode --on

# Backup files
tar czf nextcloud-backup.tar.gz ./nextcloud ./data

# Backup database
docker exec nextcloud-db pg_dump -U nextcloud nextcloud > nextcloud-db.sql

# Exit maintenance
docker exec -u www-data nextcloud php occ maintenance:mode --off
```

### Restore

```bash
# Restore files
tar xzf nextcloud-backup.tar.gz

# Restore database
docker exec -i nextcloud-db psql -U nextcloud nextcloud < nextcloud-db.sql

# Rescan files
docker exec -u www-data nextcloud php occ files:scan --all
```

## Troubleshooting

### Check Logs

```bash
docker logs nextcloud
docker exec nextcloud cat /var/www/html/data/nextcloud.log | tail -50
```

### Common Issues

**Untrusted Domain:**
```php
'trusted_domains' => [
  0 => 'localhost',
  1 => 'cloud.example.com',
],
```

**File Upload Fails:**
```php
'upload_max_filesize' => '10G',
'post_max_size' => '10G',
```

**Slow File Listing:**
```bash
docker exec -u www-data nextcloud php occ files:scan --all
docker exec -u www-data nextcloud php occ db:add-missing-indices
```

## Best Practices

1. **Use PostgreSQL** - better performance than SQLite
2. **Enable Redis** - caching and locking
3. **Run cron** - essential for background tasks
4. **Regular backups** - database + files
5. **HTTPS only** - use reverse proxy
6. **Keep updated** - security patches
7. **Limit upload size** - prevent storage issues
