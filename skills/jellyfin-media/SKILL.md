# Jellyfin Media Server

Free, open-source media server for movies, TV, and music.

## When to Use

- Stream media to any device
- Organize movie/TV libraries
- Replace Plex with self-hosted alternative
- Need hardware transcoding

## Docker Setup

### Basic Setup

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
      - /mnt/media/movies:/data/movies
      - /mnt/media/tv:/data/tvshows
      - /mnt/media/music:/data/music
    ports:
      - 8096:8096
      - 8920:8920  # HTTPS
      - 7359:7359/udp  # Discovery
      - 1900:1900/udp  # DLNA
    restart: unless-stopped
```

### With Intel QuickSync (Hardware Transcoding)

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ./config:/config
      - /mnt/media/movies:/data/movies
      - /mnt/media/tv:/data/tvshows
    devices:
      - /dev/dri:/dev/dri
    ports:
      - 8096:8096
    restart: unless-stopped
```

### With NVIDIA GPU

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./config:/config
      - /mnt/media:/data
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    ports:
      - 8096:8096
    restart: unless-stopped
```

## Library Setup

### Folder Structure

```
/mnt/media/
├── movies/
│   ├── Movie Name (2024)/
│   │   ├── Movie Name (2024).mkv
│   │   └── Movie Name (2024).srt
│   └── Another Movie (2023)/
│       └── Another Movie (2023).mp4
├── tv/
│   └── Show Name/
│       ├── Season 01/
│       │   ├── Show Name - S01E01 - Episode Title.mkv
│       │   └── Show Name - S01E02 - Episode Title.mkv
│       └── Season 02/
└── music/
    └── Artist/
        └── Album/
            ├── 01 - Track.flac
            └── cover.jpg
```

### Add Library

1. Dashboard → Libraries → Add Media Library
2. Select content type (Movies/Shows/Music)
3. Add folder path (`/data/movies`)
4. Configure metadata providers (TMDB, TVDB)
5. Set preferred language

### Metadata Providers

**Movies:**
- TheMovieDb (recommended)
- Open Media Database

**TV Shows:**
- TheTVDB
- TheMovieDb

**Music:**
- MusicBrainz

## Hardware Transcoding

### Enable in Dashboard

1. Dashboard → Playback → Transcoding
2. Hardware acceleration: Select your method
   - Intel QuickSync (Intel iGPU)
   - NVIDIA NVENC (NVIDIA GPU)
   - VAAPI (AMD/Intel Linux)
3. Enable hardware decoding for codecs

### Check GPU Access

```bash
# Intel QuickSync
ls -la /dev/dri
docker exec jellyfin ls -la /dev/dri

# NVIDIA
nvidia-smi
docker exec jellyfin nvidia-smi
```

### Transcode Settings

| Setting | Recommendation |
|---------|---------------|
| Hardware acceleration | Your GPU type |
| H.264 encoding | Hardware |
| HEVC encoding | Hardware (if supported) |
| Throttle transcodes | Disabled (for single user) |
| Max concurrent transcodes | Based on GPU |

## Remote Access

### Reverse Proxy (Recommended)

Use Traefik/Caddy for HTTPS:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.jellyfin.rule=Host(`media.example.com`)"
  - "traefik.http.routers.jellyfin.entrypoints=websecure"
  - "traefik.http.routers.jellyfin.tls.certresolver=letsencrypt"
  - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
```

### Base URL (Behind Reverse Proxy)

Dashboard → Networking:
- Base URL: `/jellyfin` (if using path-based routing)
- Secure connection mode: Handled by reverse proxy

## Users & Permissions

### Create User

1. Dashboard → Users → Add User
2. Set username and password
3. Configure permissions:
   - Library access
   - Remote access
   - Download permissions

### Parental Controls

Per-user settings:
- Block unrated content
- Maximum parental rating
- Block specific tags

## Plugins

### Recommended Plugins

1. **Intro Skipper** - Auto-skip TV intros
2. **Fanart** - Enhanced artwork
3. **Anime** - Better anime metadata
4. **Reports** - Library statistics
5. **Trakt** - Sync watch history

### Install Plugin

1. Dashboard → Plugins → Catalog
2. Find plugin
3. Install
4. Restart Jellyfin

## External Players

### Kodi Integration

1. Install Jellyfin for Kodi add-on
2. Enter server address
3. Login with Jellyfin credentials

### MPV Shim

For desktop playback without browser:
```bash
pip install jellyfin-mpv-shim
```

## Troubleshooting

### Playback Issues

```bash
# Check transcode logs
docker logs jellyfin | grep -i transcode

# Check ffmpeg processes
docker exec jellyfin ps aux | grep ffmpeg
```

### Metadata Not Fetching

1. Check internet connectivity from container
2. Verify API keys (TMDB, TVDB)
3. Check filename format matches expected pattern
4. Run library scan: Libraries → [...] → Scan Library

### GPU Not Detected

```bash
# Check device permissions
ls -la /dev/dri/

# Add user to render group (Intel)
sudo usermod -aG render jellyfin

# Check container access
docker exec jellyfin ls -la /dev/dri/
```

### Common Fixes

```bash
# Reset database
docker exec jellyfin rm /config/data/jellyfin.db

# Clear cache
docker exec jellyfin rm -rf /config/cache/*

# Fix permissions
sudo chown -R 1000:1000 /mnt/media
```

## Performance Tips

1. **Use SSD for config** - metadata/database
2. **Direct play when possible** - avoid transcoding
3. **Optimize library** - remove duplicates
4. **Enable hardware transcoding** - much faster
5. **Tune transcode settings** - match your clients

## Backup

```bash
# Backup config
docker exec jellyfin tar czf - /config > jellyfin-backup.tar.gz

# Important files
./config/
├── data/
│   └── jellyfin.db    # User data, watch history
├── config/
│   └── system.xml     # Server configuration
└── plugins/           # Installed plugins
```

## Best Practices

1. **Consistent naming** - use Jellyfin's expected format
2. **Hardware transcoding** - essential for 4K
3. **Separate metadata disk** - SSD for responsiveness
4. **Regular backups** - especially jellyfin.db
5. **Use reverse proxy** - for remote access
6. **Pin versions in production** - avoid surprise updates
