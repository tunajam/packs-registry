# Unraid Essentials

NAS OS with Docker and VM support.

## When to Use

- Mixed disk sizes in array
- Need parity protection
- Want Docker + VMs on NAS
- User-friendly NAS management

## Array Concepts

### Parity

- **Single parity**: Lose 1 disk, survive
- **Dual parity**: Lose 2 disks, survive
- Parity disk must be >= largest data disk

### Cache Pool

- SSD(s) for fast writes
- Mover transfers to array on schedule
- Can run Docker containers

### Unassigned Devices

- Disks outside the array
- Direct mount for special use cases

## Initial Setup

### Array Configuration

1. Main → Array Devices
2. Assign parity disk (largest)
3. Assign data disks
4. Start array → Format

### Cache Pool

1. Main → Cache Devices
2. Assign SSD(s)
3. Start array

For redundancy, use 2+ cache disks (mirror).

## Shares

### Create Share

Shares → Add Share:
- Name: `media`
- Primary storage: Cache (prefer)
- Secondary storage: Array
- Allocation method: High-water
- Minimum free space: 50G

### Share Settings

| Setting | Recommendation |
|---------|---------------|
| Use cache | Prefer (writes to cache, mover to array) |
| Allocation | High-water (fills disks evenly) |
| Split level | Auto (or manual for specific layouts) |

### Access Control

- Export: Yes (for SMB/NFS)
- Security: Private/Secure
- Read/Write users: Select users

## Docker

### Enable Docker

Settings → Docker:
- Enable Docker: Yes
- Docker directory: `/mnt/cache/docker`
- Docker image: 20GB+

### Community Applications

Plugins → Install → Community Applications

Then: Apps → Browse containers

### Container Setup

Apps → [Container] → Install:
- Repository: auto-filled
- Network: Bridge or Host
- Paths: Map container paths to Unraid paths

### Common Path Mappings

| Container Path | Unraid Path |
|----------------|-------------|
| `/config` | `/mnt/user/appdata/[app]` |
| `/data` | `/mnt/user/media` |
| `/downloads` | `/mnt/user/downloads` |

### Docker Compose

Install Docker Compose Manager from CA.

Path: `/boot/config/plugins/compose.manager/projects/`

## VMs

### Enable VMs

Settings → VM Manager:
- Enable VMs: Yes
- IOMMU: Enabled (requires CPU support)

### Create VM

VMs → Add VM:
1. Select OS template
2. Assign CPU cores
3. Assign RAM
4. Create/assign vdisk
5. Pass through devices (GPU, USB)

### GPU Passthrough

1. Enable IOMMU in BIOS
2. Settings → VM Manager → VFIO-PCI
3. Select GPU for passthrough
4. Assign to VM

### VM Templates

Common templates included:
- Windows 10/11
- Ubuntu
- Debian
- Custom

## Plugins

### Essential Plugins

| Plugin | Purpose |
|--------|---------|
| Community Applications | App store |
| Unassigned Devices | External disk mount |
| Dynamix File Manager | Web file browser |
| User Scripts | Scheduled scripts |
| Parity Check Tuner | Schedule parity checks |
| Fix Common Problems | Health diagnostics |

### Install Plugin

Plugins → Install Plugin → Paste URL

Or: Apps → Search plugin name

## Mover

### Schedule

Settings → Scheduler → Mover:
- Schedule: Daily at 3 AM
- Move all from cache to array

### Manual Run

Main → Array Operations → Move

### Per-Share Control

Shares → [Share] → Use cache:
- **No**: Direct to array
- **Yes**: Cache only (never move)
- **Prefer**: Cache then move
- **Only**: Only on cache

## User Scripts

### Create Script

Settings → User Scripts → Add:
```bash
#!/bin/bash
# Clean up old downloads
find /mnt/user/downloads -mtime +30 -delete
```

### Schedule Options

- Run at first array start only
- Run at array start/stop
- Custom cron schedule
- Run in background

## Network

### Static IP

Settings → Network Settings:
- Interface: eth0
- IPv4 address assignment: Static
- IP: 192.168.1.10
- Gateway: 192.168.1.1
- DNS: 192.168.1.10 (Pi-hole)

### Bonding/Link Aggregation

Settings → Network Settings:
- Enable bonding: Yes
- Bonding mode: Balance-rr or LACP

## Backup

### Flash Backup

Main → Boot Device → Flash Backup

Or: Settings → Scheduler → Backup

### Config Backup

```bash
# Important directories
/boot/config/
├── go                    # Startup script
├── plugins/              # Plugin configs
├── shares/               # Share definitions
├── docker.cfg            # Docker settings
└── super.dat             # Array config
```

### Appdata Backup

Install "Appdata Backup" from CA:
- Schedule: Weekly
- Path: `/mnt/user/backups/appdata`

## Monitoring

### Dashboard

Add to Dashboard:
- Array Status
- Docker containers
- VM status
- Temperature

### Notifications

Settings → Notification Settings:
- Email/Slack/Discord
- Array problems
- SMART errors
- Parity check results

### SMART Monitoring

Settings → Disk Settings:
- SMART polling: 30 minutes
- Temperature warning: 45°C

## Troubleshooting

### Disk Issues

```bash
# Check disk health
hdparm -I /dev/sdX

# SMART test
smartctl -t short /dev/sdX
```

### Docker Issues

```bash
# Reset Docker
Settings → Docker → Enable Docker: No
rm -rf /mnt/cache/docker
Settings → Docker → Enable Docker: Yes
```

### Parity Sync Slow

Main → Array Operations:
- Parity check runs during off-peak
- Or use Parity Check Tuner plugin

### Common Problems

1. **Disk disabled**: Check connections, run SMART test
2. **Docker container won't start**: Check paths, permissions
3. **VM won't start**: IOMMU, CPU/RAM allocation
4. **Mover not working**: Check cache share settings

## Best Practices

1. **Use cache for Docker** - better performance
2. **Regular parity checks** - monthly minimum
3. **Backup flash drive** - contains all config
4. **Monitor temperatures** - hot disks fail faster
5. **Leave array 20% free** - performance and flexibility
6. **Use appdata backup** - easy container recovery
7. **Test restores** - backups are useless if they don't work
