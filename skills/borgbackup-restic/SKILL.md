# BorgBackup & Restic Quick Reference

Deduplicating backup tools comparison and quick commands.

## Comparison

| Feature | BorgBackup | Restic |
|---------|------------|--------|
| Deduplication | Excellent | Excellent |
| Encryption | AES-256 | AES-256 |
| Cloud backends | Via rclone | Native S3, B2, etc. |
| Speed | Fast | Faster |
| Compression | Multiple options | Auto |
| Maturity | Older, proven | Newer, active |

## Restic Quick Reference

```bash
# Initialize
restic init -r /backup/repo
restic init -r b2:bucket:path

# Backup
restic -r /backup/repo backup /data
restic -r /backup/repo backup --exclude="*.tmp" /data

# List snapshots
restic -r /backup/repo snapshots

# Restore
restic -r /backup/repo restore latest --target /restore

# Prune
restic -r /backup/repo forget --keep-daily 7 --keep-weekly 4 --prune

# Check
restic -r /backup/repo check
```

### Environment Variables

```bash
export RESTIC_REPOSITORY="b2:bucket:repo"
export RESTIC_PASSWORD_FILE="/root/.restic-password"
export B2_ACCOUNT_ID="xxx"
export B2_ACCOUNT_KEY="xxx"
```

## BorgBackup Quick Reference

```bash
# Initialize
borg init --encryption=repokey /backup/repo
borg init --encryption=repokey ssh://user@server/path

# Backup
borg create /backup/repo::{now:%Y-%m-%d} /data
borg create --compression lz4 /backup/repo::backup /data

# List archives
borg list /backup/repo

# Restore
cd /restore && borg extract /backup/repo::2024-01-15

# Prune
borg prune --keep-daily=7 --keep-weekly=4 /backup/repo

# Check
borg check /backup/repo
```

### Environment Variables

```bash
export BORG_REPO="/backup/repo"
export BORG_PASSPHRASE="password"
# Or
export BORG_PASSCOMMAND="cat /root/.borg-password"
```

## Docker Wrappers

### Restic (mazzolino/restic)

```yaml
services:
  backup:
    image: mazzolino/restic
    hostname: homelab
    environment:
      - RESTIC_REPOSITORY=b2:bucket:repo
      - RESTIC_PASSWORD=secret
      - B2_ACCOUNT_ID=xxx
      - B2_ACCOUNT_KEY=xxx
      - BACKUP_CRON=0 2 * * *
      - RESTIC_FORGET_ARGS=--keep-daily 7 --keep-weekly 4
    volumes:
      - /data:/data:ro
```

### BorgBackup (borgmatic)

```yaml
services:
  borgmatic:
    image: ghcr.io/borgmatic-collective/borgmatic
    environment:
      - TZ=America/Los_Angeles
    volumes:
      - /data:/data:ro
      - ./borgmatic.yaml:/etc/borgmatic/config.yaml
      - ./borg-repo:/repo
```

## When to Use Each

**Choose Restic:**
- Cloud backup (native S3, B2)
- Simpler setup
- Better Windows support
- Multiple backup targets

**Choose Borg:**
- Local/SSH backups
- Need specific compression
- Established workflow
- Via rclone for cloud
