# Directory Structure Reference

This document explains the complete directory structure for the Jellyfin Media Stack.

## Overview

The stack uses two main directory trees:
1. **Docker Configs** (`/opt/docker/media-stack/`) - Container configurations and databases
2. **Media Storage** (`/mnt/media/`) - Your actual media files and downloads

---

## Docker Configuration Directory

```
/opt/docker/media-stack/
│
├── docker-compose.yml          # Main deployment file
├── .env                        # Environment variables (PRIVATE - never commit!)
├── .env.example                # Example env file (safe to share)
├── README.md                   # Full setup guide
├── QUICKSTART.md              # Quick reference
├── TROUBLESHOOTING.md         # Problem solving
├── DIRECTORY_STRUCTURE.md     # This file
│
├── jellyfin/
│   └── config/                # Jellyfin configuration
│       ├── data/              # Database
│       ├── metadata/          # Movie/Show metadata
│       ├── plugins/           # Jellyfin plugins
│       └── log/               # Logs
│
├── radarr/
│   └── config/                # Radarr configuration
│       ├── Backups/           # Database backups
│       ├── logs/              # Logs
│       ├── MediaCover/        # Cached posters
│       └── radarr.db          # Main database
│
├── sonarr/
│   └── config/                # Sonarr configuration
│       ├── Backups/
│       ├── logs/
│       ├── MediaCover/
│       └── sonarr.db
│
├── lidarr/
│   └── config/                # Lidarr configuration
│       ├── Backups/
│       ├── logs/
│       └── lidarr.db
│
├── readarr/
│   └── config/                # Readarr configuration
│       ├── Backups/
│       ├── logs/
│       └── readarr.db
│
├── kapowarr/
│   └── config/                # Kapowarr database
│       └── db.sqlite
│
├── prowlarr/
│   └── config/                # Prowlarr configuration
│       ├── Backups/
│       ├── logs/
│       └── prowlarr.db        # Indexer database
│
├── autobrr/
│   └── config/                # Autobrr configuration
│       ├── config.toml
│       └── autobrr.db
│
├── jellyseerr/
│   └── config/                # Jellyseerr configuration
│       ├── db/                # Request database
│       └── logs/
│
├── qbittorent/                # Note: spelled "qbittorent" in compose
│   └── config/                # qBittorrent configuration
│       ├── qBittorrent/
│       │   ├── config/
│       │   │   └── qBittorrent.conf  # Main config
│       │   └── BT_backup/     # Torrent state files (used by cross-seed)
│       └── wireguard/         # VPN configs (managed by Gluetun)
│
├── tdarr/
│   ├── server/                # Tdarr server data
│   ├── configs/               # Tdarr configuration
│   │   └── Tdarr/
│   │       ├── Configs/
│   │       └── Logs/
│   └── logs/                  # Application logs
│
├── bazarr/
│   └── config/                # Bazarr configuration
│       ├── db/                # Subtitle database
│       ├── backup/
│       └── log/
│
├── cross-seed/
│   └── config/                # Cross-seed configuration
│       └── config.js          # Manual configuration required
│
├── wizarr/
│   ├── data/
│   │   └── database/          # User invitation database
│   └── cache/                 # Application cache
│
├── unpackerr/
│   └── config/                # Unpackerr configuration
│       └── unpackerr.conf
│
├── dispatcharr/
│   └── data/                  # Dispatcharr data
│
├── homarr/                    # Dashboard
│   ├── configs/
│   ├── data/
│   ├── icons/
│   └── logs/
│
├── decluttarr/
│   # No persistent config, uses environment variables
│
├── jellystat/
│   ├── postgres-data/         # PostgreSQL database files
│   │   └── [PostgreSQL data files]
│   └── postgres-backup/       # Automatic database backups
│       └── [Backup .sql files]
│
├── janitorr/
│   ├── application.yml        # Manual configuration required
│   └── logs/                  # Application logs
│
└── profilarr/
    └── config/                # Profilarr configuration

```

**Disk Usage Estimates:**
- Fresh install: ~5GB
- After 1 month: ~20GB (depends on metadata caching)
- Jellyfin metadata can grow large with many items
- Recommend 50-100GB for `/opt/docker/`

---

## Media Storage Directory

```
/mnt/media/                    # Main media mount point
│
├── media/                     # Organized media (Jellyfin reads from here)
│   ├── movies/                # Movies library
│   │   ├── Movie Name (2024)/
│   │   │   ├── Movie Name (2024) - 1080p.mkv
│   │   │   └── Movie Name (2024) - 1080p.srt
│   │   └── Another Movie (2023)/
│   │       └── Another Movie (2023) - 2160p.mkv
│   │
│   ├── shows/                 # TV Shows library
│   │   ├── Show Name/
│   │   │   ├── Season 01/
│   │   │   │   ├── Show Name - S01E01 - Episode.mkv
│   │   │   │   └── Show Name - S01E02 - Episode.mkv
│   │   │   └── Season 02/
│   │   │       └── Show Name - S02E01 - Episode.mkv
│   │   └── Another Show/
│   │       └── Season 01/
│   │
│   ├── music/                 # Music library
│   │   ├── Artist Name/
│   │   │   ├── Album Name (2024)/
│   │   │   │   ├── 01 - Track.flac
│   │   │   │   └── 02 - Track.flac
│   │   │   └── cover.jpg
│   │   └── Various Artists/
│   │
│   ├── books/                 # Books library
│   │   ├── Author Name/
│   │   │   ├── Book Title (2024)/
│   │   │   │   └── Book Title.epub
│   │   │   └── cover.jpg
│   │   └── Another Author/
│   │
│   └── comics/                # Comics library
│       ├── Comic Series/
│       │   ├── Comic Series #001.cbz
│       │   ├── Comic Series #002.cbz
│       │   └── cover.jpg
│       └── Another Series/
│
├── torrents/                  # Torrent downloads (qBittorrent)
│   ├── complete/              # Completed downloads
│   │   ├── movies/            # Radarr downloads here
│   │   ├── shows/             # Sonarr downloads here
│   │   ├── music/             # Lidarr downloads here
│   │   ├── books/             # Readarr downloads here
│   │   └── comics/            # Kapowarr downloads here
│   │
│   ├── incomplete/            # Active downloads
│   │
│   ├── cross-seed/            # Cross-seeded torrents
│   │   └── [Auto-generated by cross-seed]
│   │
│   └── watch/                 # Watch folder (optional)
│
└── temp_downloads/            # Temporary downloads (Kapowarr)
    └── [Temporary comic downloads]

```

**How Data Flows:**
1. User requests media in Jellyseerr
2. Radarr/Sonarr finds it in Prowlarr
3. Sends to qBittorrent → downloads to `/data/torrents/incomplete/`
4. When complete → moves to `/data/torrents/complete/movies/`
5. Unpackerr extracts if needed
6. Radarr/Sonarr hardlinks/moves to `/data/media/movies/Movie Name (2024)/`
7. Jellyfin scans and shows in library

**Disk Usage Estimates:**
- Varies greatly depending on your library
- 1080p movie: ~2-8 GB
- 2160p (4K) movie: ~20-80 GB
- TV episode (1080p): ~1-3 GB
- Music album (FLAC): ~300-500 MB
- Book (EPUB): ~1-5 MB
- Comic (CBZ): ~50-200 MB

---

## Container Volume Mappings

Understanding how paths map between containers and host:

### Jellyfin
```yaml
volumes:
  - /opt/docker/media-stack/jellyfin/config:/config
  - /mnt/media/media:/media
```
- Inside container: `/media/movies` → Host: `/mnt/media/media/movies`

### Radarr
```yaml
volumes:
  - /opt/docker/media-stack/radarr/config:/config
  - /mnt/media:/data
```
- Inside container: `/data/media/movies` → Host: `/mnt/media/media/movies`
- Inside container: `/data/torrents` → Host: `/mnt/media/torrents`

### qBittorrent
```yaml
volumes:
  - /opt/docker/media-stack/qbittorent/config:/config
  - /mnt/media:/data
```
- Inside container: `/data/torrents/complete` → Host: `/mnt/media/torrents/complete`

**IMPORTANT:** All *arr apps and qBittorrent must see the same paths!
- ✅ Correct: All use `/data` mapped to `/mnt/media`
- ❌ Wrong: Different paths in different containers

---

## Path Configuration in Apps

### Radarr Root Folder
```
/data/media/movies
```

### Radarr Download Client (qBittorrent)
```
Host: gluetun (or localhost if not using VPN)
Category: movies
Directory: /data/torrents/complete/movies
```

### Sonarr Root Folder
```
/data/media/shows
```

### Jellyfin Libraries
```
Movies: /media/movies
Shows:  /media/shows
Music:  /media/music
Books:  /media/books
Comics: /media/comics
```

---

## Backup Strategy

### What to Backup

**Critical (Must backup):**
```
/opt/docker/media-stack/.env                    # Environment config
/opt/docker/media-stack/docker-compose.yml      # Stack definition
/opt/docker/media-stack/*/config/               # All app configs
```

**Important (Database backups):**
```
/opt/docker/media-stack/radarr/config/Backups/
/opt/docker/media-stack/sonarr/config/Backups/
/opt/docker/media-stack/jellystat/postgres-backup/
```

**Optional (Can rebuild):**
```
/opt/docker/media-stack/jellyfin/config/metadata/   # Re-scans
/opt/docker/media-stack/jellyfin/config/data/       # Re-scans
```

**Media (Depends on source):**
- If from private trackers: Backup everything
- If from public trackers: Can re-download
- Personal media: MUST backup

### Backup Commands

**Daily backup (configs only):**
```bash
tar -czf /mnt/media/backups/configs-$(date +%Y%m%d).tar.gz \
  /opt/docker/media-stack/.env \
  /opt/docker/media-stack/docker-compose.yml \
  /opt/docker/media-stack/*/config/
```

**Weekly backup (everything):**
```bash
tar -czf /mnt/media/backups/full-backup-$(date +%Y%m%d).tar.gz \
  /opt/docker/media-stack/
```

**Exclude large cache files:**
```bash
tar -czf /mnt/media/backups/configs-$(date +%Y%m%d).tar.gz \
  --exclude='*/cache/*' \
  --exclude='*/logs/*' \
  --exclude='*/metadata/*' \
  /opt/docker/media-stack/
```

---

## Disk Space Management

### Monitor Space
```bash
# Overall
df -h

# Per service
du -sh /opt/docker/media-stack/*/ | sort -h

# Media breakdown
du -sh /mnt/media/*/ | sort -h

# Largest folders
du -h /mnt/media/ | sort -h | tail -20
```

### Clean Up Space

**Docker cleanup:**
```bash
# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune

# Remove everything unused
docker system prune -a --volumes
```

**Media cleanup:**
- Use Decluttarr (auto-removes failed downloads)
- Use Janitorr (removes watched media after X days)
- Tdarr (convert to H.265 to save 30-50% space)

**Transcode cache:**
```bash
# Clear Tdarr cache
rm -rf /transcode_cache/*

# Clear Jellyfin transcode
docker exec jellyfin rm -rf /config/data/transcodes/*
```

---

## Permission Reference

### Correct Permissions
```bash
# Configs
chown -R 1000:1000 /opt/docker/media-stack/
chmod -R 755 /opt/docker/media-stack/

# Media
chown -R 1000:1000 /mnt/media/
chmod -R 755 /mnt/media/

# Files should be 644
chmod 644 /mnt/media/media/movies/*/*.mkv

# Folders should be 755
chmod 755 /mnt/media/media/movies/*/
```

### Check Permissions
```bash
# See ownership
ls -la /mnt/media/

# See PUID/PGID
id

# See what container is using
docker exec jellyfin id
```

---

## Network Port Reference

| Service | Port | Access | VPN? |
|---------|------|--------|------|
| Jellyfin | 8096 | Web UI | No |
| Radarr | 7878 | Web UI | No |
| Sonarr | 8989 | Web UI | No |
| Lidarr | 8686 | Web UI | No |
| Readarr | 8787 | Web UI | No |
| Prowlarr | 9696 | Web UI (via Gluetun) | Yes |
| qBittorrent | 8080 | Web UI (via Gluetun) | Yes |
| qBittorrent | 8694 | Torrent port | Yes |
| Jellyseerr | 5055 | Web UI | No |
| Kapowarr | 5757 | Web UI (via Gluetun) | Yes |
| Autobrr | 7474 | Web UI (via Gluetun) | Yes |
| FlareSolverr | 8191 | API (via Gluetun) | Yes |
| Tdarr | 8265 | Web UI | No |
| Tdarr | 8266 | Server | No |
| Bazarr | 6767 | Web UI | No |
| Cross-seed | 2468 | API (via Gluetun) | Yes |
| Wizarr | 5690 | Web UI | No |
| Homarr | 7575 | Web UI | No |
| Dozzle | 9999 | Web UI | No |
| Jellystat | 3000 | Web UI | No |
| Profilarr | 6868 | Web UI | No |
| Dispatcharr | 9191 | API (via Gluetun) | Yes |

**VPN Services Access:**
- Services marked "Yes" for VPN are accessed through Gluetun's network
- Their ports are exposed on the VM's IP address
- Example: qBittorrent (via VPN) → http://VM_IP:8080

---

Generated with Claude Code - https://claude.com/claude-code
