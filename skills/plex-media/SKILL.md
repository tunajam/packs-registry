# Plex Media Server

Stream your media anywhere with Plex.

## When to Use

- Want polished streaming experience
- Need Plex-specific features (Plexamp, Plex Pass)
- Remote streaming to mobile
- Prefer managed apps over web UI

## Plex vs Jellyfin

| Feature | Plex | Jellyfin |
|---------|------|----------|
| Price | Free + Pass ($5/mo) | Free |
| Apps | Polished | Improving |
| Remote | Built-in | Manual setup |
| Transcoding | Excellent | Good |
| Privacy | Phone home | Self-hosted |

## Docker Setup

```yaml
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - VERSION=docker
      - PLEX_CLAIM=claim-xxxx  # Get from plex.tv/claim
    volumes:
      - ./config:/config
      - /mnt/media/movies:/movies
      - /mnt/media/tv:/tv
      - /mnt/media/music:/music
    devices:
      - /dev/dri:/dev/dri  # Intel QuickSync
    restart: unless-stopped
```

### Bridge Mode (Non-Host Network)

```yaml
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - VERSION=docker
    volumes:
      - ./config:/config
      - /mnt/media:/media
    ports:
      - 32400:32400
      - 1900:1900/udp
      - 5353:5353/udp
      - 8324:8324
      - 32410:32410/udp
      - 32412:32412/udp
      - 32413:32413/udp
      - 32414:32414/udp
      - 32469:32469
    restart: unless-stopped
```

## Initial Setup

### Claim Server

1. Get claim token: https://plex.tv/claim
2. Set `PLEX_CLAIM` environment variable
3. Start container
4. Access: `http://your-ip:32400/web`

### Library Setup

Settings → Libraries → Add:
- Type: Movies/TV Shows/Music
- Folders: /movies, /tv, /music
- Scanner: Plex default
- Agent: Plex default

## Hardware Transcoding

### Plex Pass Required

Hardware transcoding requires Plex Pass subscription.

### Intel QuickSync

```yaml
devices:
  - /dev/dri:/dev/dri
```

Settings → Transcoder:
- Use hardware acceleration when available: ✓

### NVIDIA GPU

```yaml
runtime: nvidia
environment:
  - NVIDIA_VISIBLE_DEVICES=all
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```

### Verify Hardware Transcoding

Check Plex dashboard during playback:
- Should show "(hw)" next to video codec

## Transcoding Settings

Settings → Transcoder:

| Setting | Recommendation |
|---------|---------------|
| Transcoder quality | Automatic |
| Background transcoding | Faster |
| HDR tone mapping | On (if supported) |
| Hardware-accelerated streaming | On |
| Maximum simultaneous transcodes | Based on CPU/GPU |

## Remote Access

### Automatic (Default)

Settings → Remote Access:
- Enable remote access
- UPnP: Automatic

### Manual Port Forward

If UPnP fails:
1. Forward port 32400 TCP on router
2. Set manual public port in Plex
3. Test connection

### Reverse Proxy (Traefik/Caddy)

```yaml
# Traefik labels
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.plex.rule=Host(`plex.example.com`)"
  - "traefik.http.routers.plex.entrypoints=websecure"
  - "traefik.http.services.plex.loadbalancer.server.port=32400"
```

Custom server access URL:
Settings → Network → Custom server access URLs:
`https://plex.example.com:443`

## Optimization

### Direct Play/Stream

Prefer direct play for best quality:
- Client: Settings → Quality → Maximum
- Automatic quality disabled

### Scheduled Tasks

Settings → Scheduled Tasks:
- Library scan: Daily
- Optimize database: Weekly
- Empty trash: After each scan

### Media Analyzer

Analyze media to generate previews:
Library → [...] → Analyze

## Tautulli (Monitoring)

```yaml
services:
  tautulli:
    image: lscr.io/linuxserver/tautulli:latest
    container_name: tautulli
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
    ports:
      - 8181:8181
    restart: unless-stopped
```

Features:
- Viewing history
- User statistics
- Notifications
- Graphs and charts

## Overseerr (Requests)

```yaml
services:
  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
    ports:
      - 5055:5055
    restart: unless-stopped
```

Features:
- Media request management
- Integration with Sonarr/Radarr
- User-friendly request interface

## Folder Structure

```
/mnt/media/
├── movies/
│   └── Movie Name (2024)/
│       ├── Movie Name (2024).mkv
│       ├── Movie Name (2024).srt
│       └── poster.jpg
├── tv/
│   └── Show Name/
│       ├── poster.jpg
│       ├── Season 01/
│       │   ├── Show Name - S01E01.mkv
│       │   └── Show Name - S01E02.mkv
│       └── Season 02/
└── music/
    └── Artist/
        └── Album (2024)/
            ├── 01 - Track.flac
            └── cover.jpg
```

## Troubleshooting

### Buffering Issues

1. Check transcoding (use direct play if possible)
2. Verify network bandwidth
3. Reduce quality settings
4. Check storage I/O

### Library Not Scanning

```bash
# Check permissions
ls -la /mnt/media

# Ensure PUID/PGID can access
chown -R 1000:1000 /mnt/media
```

### Remote Access Not Working

1. Check UPnP on router
2. Manually forward port 32400
3. Verify firewall allows traffic
4. Check Plex status page

### Transcoding Fails

1. Verify hardware acceleration enabled
2. Check GPU drivers
3. Review Plex transcoder logs:
```bash
cat config/Library/Application\ Support/Plex\ Media\ Server/Logs/Plex\ Transcoder*.log
```

## Backup

### Important Paths

```
./config/Library/Application Support/Plex Media Server/
├── Preferences.xml          # Server settings
├── Plug-in Support/
│   └── Databases/           # Media database
└── Metadata/                # Posters, artwork
```

### Backup Script

```bash
#!/bin/bash
docker stop plex
tar czf plex-backup-$(date +%Y%m%d).tar.gz ./config
docker start plex
```

## Best Practices

1. **Use direct play** - best quality, no CPU load
2. **Hardware transcode** - Intel QuickSync is excellent
3. **Proper naming** - Plex matches easier
4. **Network mode: host** - simplest for local access
5. **Plex Pass for hardware** - transcoding requires it
6. **Monitor with Tautulli** - track usage
7. **Regular backups** - especially Preferences.xml
