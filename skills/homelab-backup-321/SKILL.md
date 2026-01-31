# Homelab Backup Strategy

The 3-2-1 rule and tools to implement it.

## The 3-2-1 Rule

- **3** copies of your data
- **2** different storage media/types
- **1** offsite copy

Example:
1. Original on NAS
2. Local backup to external drive
3. Cloud backup to Backblaze B2

## Backup Tools Comparison

| Tool | Deduplication | Encryption | Cloud Support | Complexity |
|------|---------------|------------|---------------|------------|
| Restic | Excellent | Yes | Native | Low |
| Borg | Excellent | Yes | Via rclone | Medium |
| Rclone | None | Optional | Native | Low |
| Duplicati | Yes | Yes | Native | Low |

## Restic

### Install

```bash
# Ubuntu/Debian
apt install restic

# macOS
brew install restic
```

### Initialize Repository

```bash
# Local
restic init --repo /mnt/backup/restic-repo

# Backblaze B2
export B2_ACCOUNT_ID="your-account-id"
export B2_ACCOUNT_KEY="your-account-key"
restic init --repo b2:bucket-name:restic-repo

# S3/MinIO
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
restic init --repo s3:s3.amazonaws.com/bucket-name/restic-repo
```

### Create Backup

```bash
# Basic backup
restic -r /mnt/backup/restic-repo backup /home/user /etc

# With excludes
restic -r /mnt/backup/restic-repo backup \
  --exclude="*.tmp" \
  --exclude-caches \
  --exclude=".cache" \
  /home/user

# Tag backup
restic -r /mnt/backup/restic-repo backup --tag daily /data
```

### List Snapshots

```bash
restic -r /mnt/backup/restic-repo snapshots
```

### Restore

```bash
# Restore entire snapshot
restic -r /mnt/backup/restic-repo restore latest --target /restore

# Restore specific files
restic -r /mnt/backup/restic-repo restore latest --target /restore --include "/home/user/documents"
```

### Retention Policy

```bash
# Keep last 7 daily, 4 weekly, 12 monthly
restic -r /mnt/backup/restic-repo forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune
```

### Automated Backup Script

```bash
#!/bin/bash
set -euo pipefail

export RESTIC_REPOSITORY="b2:mybucket:restic"
export RESTIC_PASSWORD_FILE="/root/.restic-password"
export B2_ACCOUNT_ID="your-id"
export B2_ACCOUNT_KEY="your-key"

# Backup
restic backup \
  --exclude-caches \
  --exclude="*.tmp" \
  /home /etc /var/lib/docker/volumes

# Cleanup old snapshots
restic forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune

# Check repository health
restic check
```

### Docker Integration

```yaml
services:
  restic-backup:
    image: mazzolino/restic
    container_name: restic-backup
    hostname: homelab
    environment:
      - TZ=America/Los_Angeles
      - RESTIC_REPOSITORY=b2:mybucket:restic
      - RESTIC_PASSWORD=your-password
      - B2_ACCOUNT_ID=your-id
      - B2_ACCOUNT_KEY=your-key
      - BACKUP_CRON=0 2 * * *
      - RESTIC_FORGET_ARGS=--keep-daily 7 --keep-weekly 4 --keep-monthly 12
    volumes:
      - /home:/data/home:ro
      - /var/lib/docker/volumes:/data/docker-volumes:ro
    restart: unless-stopped
```

## BorgBackup

### Initialize

```bash
# Local
borg init --encryption=repokey /mnt/backup/borg-repo

# Remote (SSH)
borg init --encryption=repokey ssh://user@server/path/to/repo
```

### Create Backup

```bash
borg create \
  --stats --progress \
  --compression lz4 \
  /mnt/backup/borg-repo::'{hostname}-{now:%Y-%m-%d}' \
  /home /etc

# With excludes
borg create \
  --exclude '*.tmp' \
  --exclude 'sh:**/.cache' \
  /mnt/backup/borg-repo::{now:%Y-%m-%d} \
  /data
```

### List Archives

```bash
borg list /mnt/backup/borg-repo
```

### Restore

```bash
# List files in archive
borg list /mnt/backup/borg-repo::2024-01-15

# Extract
cd /restore
borg extract /mnt/backup/borg-repo::2024-01-15

# Extract specific paths
borg extract /mnt/backup/borg-repo::2024-01-15 home/user/documents
```

### Prune

```bash
borg prune \
  --keep-daily=7 \
  --keep-weekly=4 \
  --keep-monthly=12 \
  /mnt/backup/borg-repo
```

### Borg + Rclone (Cloud Sync)

```bash
# Mount borg repo
borg mount /mnt/backup/borg-repo /mnt/borg-mount

# Sync to cloud with rclone
rclone sync /mnt/backup/borg-repo remote:backup/borg
```

## Rclone

### Configure Remote

```bash
rclone config
# Interactive setup for S3, B2, Google Drive, etc.
```

### Sync Operations

```bash
# Sync local to remote
rclone sync /local/path remote:bucket/path

# Sync with delete protection
rclone sync /local/path remote:bucket/path --backup-dir remote:bucket/deleted

# Copy only (no delete)
rclone copy /local/path remote:bucket/path

# With bandwidth limit
rclone sync /local/path remote:bucket/path --bwlimit 10M
```

### Mount Cloud Storage

```bash
# Mount as filesystem
rclone mount remote:bucket /mnt/cloud --daemon --vfs-cache-mode full

# Add to fstab
remote:bucket /mnt/cloud rclone rw,noauto,x-systemd.automount,args2env,config=/root/.config/rclone/rclone.conf 0 0
```

### Encrypted Remote

```bash
# Create encrypted remote (in rclone config)
rclone config
# Choose "crypt" and point to existing remote
```

## Docker Volume Backup

### Using Restic

```bash
#!/bin/bash
# backup-docker-volumes.sh

VOLUMES=$(docker volume ls -q)

for volume in $VOLUMES; do
  docker run --rm \
    -v $volume:/data:ro \
    -v /mnt/backup:/backup \
    restic/restic \
    -r /backup/docker-volumes backup /data --tag $volume
done
```

### Using docker-volume-backup

```yaml
services:
  backup:
    image: offen/docker-volume-backup:latest
    environment:
      - BACKUP_CRON_EXPRESSION=0 2 * * *
      - BACKUP_FILENAME=backup-%Y-%m-%d.tar.gz
      - AWS_S3_BUCKET_NAME=my-backup-bucket
      - AWS_ACCESS_KEY_ID=your-key
      - AWS_SECRET_ACCESS_KEY=your-secret
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - my-app-data:/backup/my-app-data:ro
    restart: unless-stopped
```

## Database Backups

### PostgreSQL

```bash
# Dump
docker exec postgres pg_dump -U user database > backup.sql

# Restore
docker exec -i postgres psql -U user database < backup.sql
```

### MySQL/MariaDB

```bash
# Dump
docker exec mysql mysqldump -u user -ppassword database > backup.sql

# Restore
docker exec -i mysql mysql -u user -ppassword database < backup.sql
```

### Automated DB + Files

```bash
#!/bin/bash
BACKUP_DIR=/mnt/backup/$(date +%Y-%m-%d)
mkdir -p $BACKUP_DIR

# Database
docker exec postgres pg_dump -U nextcloud nextcloud | gzip > $BACKUP_DIR/nextcloud-db.sql.gz

# Files
restic -r /mnt/backup/restic backup $BACKUP_DIR /data/nextcloud

# Cleanup local
rm -rf $BACKUP_DIR
```

## Monitoring Backups

### Healthchecks.io

```bash
# At end of backup script
curl -fsS -m 10 --retry 5 https://hc-ping.com/your-uuid
```

### Uptime Kuma

Add push monitor:
```bash
curl "http://uptime-kuma:3001/api/push/YOUR_TOKEN?status=up&msg=Backup%20complete"
```

## Verification

```bash
# Restic check
restic -r /mnt/backup/restic-repo check

# Verify specific snapshot
restic -r /mnt/backup/restic-repo check --read-data-subset=5%

# Borg check
borg check /mnt/backup/borg-repo
borg check --verify-data /mnt/backup/borg-repo
```

## Best Practices

1. **Test restores** - backups are useless if you can't restore
2. **Encrypt everything** - especially offsite backups
3. **Automate** - manual backups get forgotten
4. **Monitor** - alert on backup failures
5. **Version retention** - keep multiple versions
6. **Document** - write down restore procedures
7. **Separate credentials** - backup keys != daily use keys
