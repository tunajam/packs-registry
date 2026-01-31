# TrueNAS and ZFS Storage

Enterprise-grade storage for homelab using TrueNAS and ZFS.

## When to Use

- Need reliable NAS storage
- Want ZFS data integrity (checksums, snapshots)
- Sharing files via SMB/NFS
- Building a media server backend

## ZFS Concepts

### Pool Layouts (vdevs)

| Layout | Disks | Usable | Fault Tolerance |
|--------|-------|--------|-----------------|
| Stripe | 2+ | 100% | None |
| Mirror | 2 | 50% | 1 disk |
| RAIDZ1 | 3+ | n-1 | 1 disk |
| RAIDZ2 | 4+ | n-2 | 2 disks |
| RAIDZ3 | 5+ | n-3 | 3 disks |

**Recommendation:** RAIDZ1 for 3-4 disks, RAIDZ2 for 5+ disks.

### Create Pool (CLI)

```bash
# Mirror (2 disks)
zpool create tank mirror /dev/sda /dev/sdb

# RAIDZ1 (3+ disks)
zpool create tank raidz /dev/sda /dev/sdb /dev/sdc

# RAIDZ2 (4+ disks)
zpool create tank raidz2 /dev/sda /dev/sdb /dev/sdc /dev/sdd

# With special vdev for metadata (SSD)
zpool create tank raidz2 /dev/sd{a,b,c,d} special mirror /dev/nvme0n1 /dev/nvme1n1
```

### Pool Status

```bash
zpool status
zpool list
zpool iostat -v 5  # Live I/O stats
```

## Datasets

### Create Datasets

```bash
# Create dataset
zfs create tank/media
zfs create tank/media/movies
zfs create tank/media/tv
zfs create tank/backups

# Set properties
zfs set compression=lz4 tank/media
zfs set recordsize=1M tank/media  # Large files
zfs set atime=off tank/media      # Performance
```

### Dataset Properties

```bash
# List all properties
zfs get all tank/media

# Common properties
zfs set quota=2T tank/media/movies      # Limit size
zfs set reservation=500G tank/important # Guarantee space
zfs set copies=2 tank/critical          # Extra redundancy
```

## Snapshots

### Manual Snapshots

```bash
# Create snapshot
zfs snapshot tank/media@2024-01-15

# Recursive snapshot (all child datasets)
zfs snapshot -r tank/media@backup

# List snapshots
zfs list -t snapshot

# Rollback
zfs rollback tank/media@2024-01-15
```

### Access Snapshots (Hidden .zfs Directory)

```bash
# Enable visibility
zfs set snapdir=visible tank/media

# Access previous versions
ls /mnt/tank/media/.zfs/snapshot/2024-01-15/
```

### Automated Snapshots (TrueNAS)

Data Protection > Periodic Snapshot Tasks:
- Dataset: tank/media
- Recursive: Yes
- Lifetime: 2 weeks
- Schedule: Daily

## Replication

### Local Replication

```bash
# Send snapshot to another pool
zfs send tank/media@backup | zfs receive backup/media
```

### Remote Replication

```bash
# Initial full send
zfs send -R tank/media@backup | ssh remote zfs receive backup/media

# Incremental send
zfs send -i tank/media@old tank/media@new | ssh remote zfs receive backup/media
```

### TrueNAS Replication Task

Data Protection > Replication Tasks:
- Source: tank/media
- Destination: SSH + pool path
- Schedule: Daily
- Replication Mode: Incremental

## SMB Shares

### TrueNAS UI

Shares > Windows (SMB) > Add:
- Path: /mnt/tank/media
- Name: media
- Purpose: Multi-user time machine (or Default)

### Permissions

Datasets > tank/media > Edit Permissions:
- User: your-user
- Group: your-group
- Apply recursively

### ACLs for Mixed Access

```bash
# TrueNAS shell
midclt call sharing.smb.create '{
  "path": "/mnt/tank/shared",
  "name": "shared",
  "purpose": "DEFAULT_SHARE"
}'
```

## NFS Shares

### Create NFS Export

Shares > Unix (NFS) > Add:
- Path: /mnt/tank/media
- Maproot User: root
- Maproot Group: wheel
- Networks: 192.168.1.0/24

### Client Mount

```bash
# Linux client
mount -t nfs truenas.local:/mnt/tank/media /mnt/media

# fstab entry
truenas.local:/mnt/tank/media /mnt/media nfs defaults,_netdev 0 0
```

## Performance Tuning

### ARC (Adaptive Replacement Cache)

```bash
# Check ARC stats
arc_summary

# In TrueNAS, ARC is auto-tuned based on RAM
# Minimum 8GB RAM, 1GB per TB of storage recommended
```

### L2ARC (SSD Read Cache)

```bash
# Add L2ARC device
zpool add tank cache /dev/nvme0n1

# Best for random read workloads
# Not useful for sequential (media streaming)
```

### SLOG (ZIL Device)

```bash
# Add SLOG for sync writes
zpool add tank log mirror /dev/nvme0n1 /dev/nvme1n1

# Important for VMs, databases
# Use enterprise SSDs with power loss protection
```

### Special vdev (Metadata)

```bash
# For datasets with many small files
zpool add tank special mirror /dev/nvme0n1 /dev/nvme1n1
zfs set special_small_blocks=128K tank/files
```

## Scrub and Maintenance

### Schedule Scrub

```bash
# Manual scrub
zpool scrub tank

# TrueNAS: Data Protection > Scrub Tasks
# Weekly scrub recommended
```

### Replace Failed Disk

```bash
# Check status
zpool status tank

# Replace disk
zpool replace tank /dev/sda /dev/sde

# Monitor rebuild
zpool status tank
```

## Common Commands

```bash
# Pool info
zpool status tank
zpool list -v tank
zpool history tank

# Dataset info
zfs list
zfs list -r tank/media
zfs get all tank/media

# Snapshots
zfs list -t snapshot
zfs destroy tank/media@old-snapshot

# Check space usage
zfs list -o name,used,avail,refer,mountpoint

# Compression ratio
zfs get compressratio tank
```

## Best Practices

1. **ECC RAM required** - ZFS needs it for data integrity
2. **Never use hardware RAID** - ZFS needs raw disks
3. **Mirror boot drive** - protect your config
4. **Regular scrubs** - weekly catches bit rot
5. **Snapshot before changes** - easy rollback
6. **3-2-1 backup rule** - replication isn't backup
7. **Leave 20% free space** - ZFS performance degrades when full
8. **Use compression** - LZ4 is free performance
