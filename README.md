# Complete Jellyfin Media Stack Setup Guide for Proxmox

## Table of Contents
- [Overview](#overview)
- [Services Included](#services-included)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Part 1: Proxmox VM Setup](#part-1-proxmox-vm-setup)
- [Part 2: Docker Installation](#part-2-docker-installation)
- [Part 3: VPN Configuration (Mullvad)](#part-3-vpn-configuration-mullvad)
- [Part 4: Environment Preparation](#part-4-environment-preparation)
- [Part 5: Deploy the Stack](#part-5-deploy-the-stack)
- [Part 6: Initial Configuration](#part-6-initial-configuration)
- [Part 7: Network and External Access](#part-7-network-and-external-access)
- [Verification and Testing](#verification-and-testing)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

---

## Overview

This is a comprehensive, production-ready media server stack that provides:
- **Media Streaming**: Jellyfin for movies, TV shows, music, books, and comics
- **Automated Downloads**: Radarr (movies), Sonarr (TV), Lidarr (music), Readarr (books), Kapowarr (comics)
- **Torrent Management**: qBittorrent behind VPN (Gluetun)
- **Privacy**: All download traffic routed through Mullvad VPN
- **User Management**: Jellyseerr for requests, Wizarr for invitations
- **Monitoring**: Jellystat, Homarr dashboard, Dozzle for logs
- **Optimization**: Tdarr for transcoding, Bazarr for subtitles
- **Automation**: Decluttarr, Janitorr for cleanup, Unpackerr for archives

---

## Services Included

### Core Media Services
- **Jellyfin**: Media server for streaming content
- **Jellyseerr**: User request management
- **Jellystat**: Statistics and monitoring
- **Wizarr**: User invitation system

### Download Management (*arr stack)
- **Radarr**: Automatic movie downloads
- **Sonarr**: Automatic TV show downloads
- **Lidarr**: Music downloads
- **Readarr**: Book downloads
- **Kapowarr**: Comic downloads
- **Prowlarr**: Indexer manager (routes through VPN)
- **Autobrr**: IRC announce grabber for faster seeding

### Download Client
- **qBittorrent**: Torrent client (routes through VPN)

### VPN & Security
- **Gluetun**: VPN client (Mullvad WireGuard)
- **FlareSolverr**: Cloudflare bypass proxy

### Optimization & Utilities
- **Tdarr**: Video transcoding to save space
- **Bazarr**: Subtitle downloads
- **Cross-seed**: Cross-seeding between trackers
- **Unpackerr**: Automatic archive extraction
- **Decluttarr**: Cleanup failed/stalled downloads
- **Janitorr**: Media library cleanup
- **Profilarr**: Custom format profiles

### Monitoring & Management
- **Homarr**: Unified dashboard
- **Dozzle**: Real-time log viewer
- **Dispatcharr**: Dispatch service

---

## Architecture

```
Internet
    ↓
Mullvad VPN (Gluetun Container)
    ↓
├── qBittorrent (downloads)
├── Prowlarr (indexers)
├── Autobrr (IRC)
├── FlareSolverr (Cloudflare bypass)
├── Kapowarr (comics)
└── Dispatcharr

Jellyfin ← Media Files → *arr Stack (Sonarr, Radarr, etc.)
                              ↓
                         qBittorrent (via Gluetun)
```

**Network Design:**
- Services requiring VPN use `network_mode: "service:gluetun"`
- Other services use bridged networks (proxy, starr, jellystat)
- External networks must be created before deployment

---

## Prerequisites

### Hardware Requirements (Recommended)
- **CPU**: 4+ cores (8+ for Tdarr transcoding)
- **RAM**: 16GB minimum (32GB recommended)
- **Storage**:
  - 50GB for Docker containers/configs
  - Large storage pool for media (NFS/SMB share or local disks)
- **GPU** (Optional): Intel QuickSync or NVIDIA GPU for hardware transcoding

### Software Requirements
- Proxmox VE 8.0+ (latest version recommended)
- Mullvad VPN account (or modify for your VPN provider)
- Basic networking knowledge
- Domain name (optional, for reverse proxy)

### Network Requirements
- Static IP for the VM
- Port forwarding capability on your router (for qBittorrent seeding)
- Access to network shares (if using NAS for media storage)

---

## Part 1: Proxmox VM Setup

### Step 1.1: Create Ubuntu VM

**Why Ubuntu VM instead of LXC or Proxmox host?**
- VMs provide better isolation and stability
- Easier to backup and restore
- No risk of breaking Proxmox host
- Better Docker compatibility
- Live migration capability

**Create the VM:**

1. In Proxmox web interface, click **Create VM**

2. **General tab:**
   - VM ID: `100` (or your choice)
   - Name: `media-server`

3. **OS tab:**
   - ISO image: Ubuntu Server 24.04 LTS (download from Ubuntu website)
   - Type: Linux
   - Version: 6.x - 2.6 Kernel

4. **System tab:**
   - Machine: q35
   - BIOS: OVMF (UEFI)
   - Add EFI Disk: Yes
   - SCSI Controller: VirtIO SCSI single

5. **Disks tab:**
   - Bus/Device: SCSI
   - Storage: local-lvm (or your preferred storage)
   - Disk size: 50GB minimum (100GB recommended for configs)
   - Cache: Write back
   - Discard: Yes (for SSD trim)
   - SSD emulation: Yes (if on SSD)

6. **CPU tab:**
   - Sockets: 1
   - Cores: 4 minimum (8+ recommended)
   - Type: host (for best performance)

7. **Memory tab:**
   - Memory: 16384 MB (16GB minimum)
   - Ballooning: No

8. **Network tab:**
   - Bridge: vmbr0
   - Model: VirtIO (paravirtualized)
   - MAC address: auto

9. Click **Finish** (do NOT start yet)

### Step 1.2: Configure Hardware Passthrough (Optional but Recommended)

**For Intel QuickSync (hardware transcoding):**

1. Edit VM hardware:
   ```bash
   # On Proxmox host shell
   qm set 100 -hostpci0 0000:00:02,rombar=0
   ```

2. Enable IOMMU in Proxmox:
   ```bash
   nano /etc/default/grub
   # Add to GRUB_CMDLINE_LINUX_DEFAULT:
   # For Intel: intel_iommu=on iommu=pt
   # For AMD: amd_iommu=on iommu=pt

   update-grub
   reboot
   ```

**For NVIDIA GPU:**
- Follow NVIDIA GPU passthrough guide (more complex, outside scope of this guide)

### Step 1.3: Install Ubuntu Server

1. Start the VM and open console
2. Follow Ubuntu installation wizard:
   - Language: English
   - Keyboard: Your layout
   - Network: Configure static IP (recommended)
     - IP: `192.168.1.100` (example - use your network)
     - Gateway: `192.168.1.1` (your router)
     - DNS: `1.1.1.1,8.8.8.8`
   - Proxy: Leave empty
   - Mirror: Default
   - Storage: Use entire disk
   - Profile setup:
     - Name: `mediaserver`
     - Server name: `media-server`
     - Username: `mediaserver`
     - Password: (strong password)
   - SSH: Install OpenSSH server ✓
   - Featured snaps: None (skip)

3. Wait for installation to complete and reboot

4. Login via SSH (recommended over console):
   ```bash
   ssh mediaserver@192.168.1.100
   ```

### Step 1.4: Update Ubuntu and Install Prerequisites

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y \
    curl \
    wget \
    git \
    nano \
    htop \
    net-tools \
    ca-certificates \
    gnupg \
    lsb-release \
    software-properties-common

# Install hardware transcoding drivers (Intel QuickSync)
# Only if you passed through GPU
sudo apt install -y \
    intel-media-va-driver-non-free \
    vainfo

# Verify GPU access (if passed through)
ls -la /dev/dri
# Should show renderD128 and card0

# Add your user to video and render groups
sudo usermod -aG video,render $USER
```

### Step 1.5: Configure Storage for Media

**Option A: Mount NFS Share (Recommended for NAS)**

```bash
# Install NFS client
sudo apt install -y nfs-common

# Create mount point
sudo mkdir -p /mnt/media

# Add to /etc/fstab for automatic mounting
sudo nano /etc/fstab

# Add this line (adjust to your NAS):
# 192.168.1.10:/volume1/media /mnt/media nfs defaults,_netdev 0 0

# Test mount
sudo mount -a

# Verify
df -h | grep media
```

**Option B: Use Local Storage**

```bash
# If using local disks, add them in Proxmox hardware
# Then format and mount inside the VM
sudo mkdir -p /mnt/media
```

**Option C: Mount SMB/CIFS Share**

```bash
# Install CIFS client
sudo apt install -y cifs-utils

# Create credentials file
sudo nano /root/.smbcredentials
# Add:
# username=your_nas_user
# password=your_nas_password

sudo chmod 600 /root/.smbcredentials

# Add to /etc/fstab:
# //192.168.1.10/media /mnt/media cifs credentials=/root/.smbcredentials,uid=1000,gid=1000 0 0

sudo mount -a
```

---

## Part 2: Docker Installation

### Step 2.1: Install Docker (Official Method)

```bash
# Remove old Docker versions if any
sudo apt remove -y docker docker-engine docker.io containerd runc

# Set up Docker repository
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
docker --version
docker compose version

# Add your user to docker group
sudo usermod -aG docker $USER

# Apply group changes (logout/login or run):
newgrp docker

# Test Docker
docker run hello-world
```

### Step 2.2: Configure Docker

```bash
# Create Docker daemon config for better logging and performance
sudo nano /etc/docker/daemon.json
```

Add this content:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

```bash
# Restart Docker
sudo systemctl restart docker
sudo systemctl enable docker

# Verify
sudo systemctl status docker
```

---

## Part 3: VPN Configuration (Mullvad)

### Step 3.1: Sign Up for Mullvad VPN

1. Go to https://mullvad.net
2. Click **Generate account**
3. Save your account number (16 digits)
4. Add time to your account (minimum 5 EUR)

### Step 3.2: Generate WireGuard Configuration

1. Go to https://mullvad.net/en/account/wireguard-config
2. Login with your account number
3. Click **Generate key**
4. Select platform: **Linux**
5. Select server location (e.g., "Amsterdam, Netherlands")
6. Click **Download file**

### Step 3.3: Extract WireGuard Credentials

Open the downloaded `.conf` file and extract:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY_HERE        # ← Copy this
Address = 10.x.x.x/32, fc00::/128         # ← Copy IPv4 address (10.x.x.x/32)
DNS = 10.64.0.1

[Peer]
PublicKey = ...
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = xxx.mullvad.net:51820
```

**Save these values:**
- `WIREGUARD_PRIVATE_KEY`: The PrivateKey value
- `WIREGUARD_ADDRESSES`: The IPv4 Address (e.g., `10.68.123.45/32`)
- Server city name (from the config filename or your selection)

---

## Part 4: Environment Preparation

### Step 4.1: Create Project Directory

```bash
# Create base directory for all configs
sudo mkdir -p /opt/docker/media-stack
cd /opt/docker/media-stack

# Set ownership
sudo chown -R $USER:$USER /opt/docker
```

### Step 4.2: Download This Repository

```bash
# Clone or download the project
git clone <repository-url> /opt/docker/media-stack
# OR create files manually using this guide
```

### Step 4.3: Create Docker Networks

The compose file uses external networks that must be created first:

```bash
docker network create proxy
docker network create jellyfin_default
docker network create starr
docker network create jellystat
docker network create gluetun_network --subnet=172.20.0.0/16
```

**Verify networks:**
```bash
docker network ls
```

### Step 4.4: Create Directory Structure

```bash
# Create all required directories
mkdir -p /opt/docker/media-stack/{jellyfin,radarr,sonarr,readarr,lidarr,kapowarr,prowlarr,autobrr,jellyseerr,qbittorent,tdarr,bazarr,cross-seed,wizarr,unpackerr,dispatcharr,homarr,decluttarr,jellystat,janitorr,profilarr}/config

mkdir -p /opt/docker/media-stack/tdarr/{server,configs,logs}
mkdir -p /opt/docker/media-stack/flaresolverr
mkdir -p /opt/docker/media-stack/jellystat/postgres-{data,backup}
mkdir -p /opt/docker/media-stack/wizarr/{data/database,cache}

# Create transcode cache directory (optional, for faster transcoding)
sudo mkdir -p /transcode_cache
sudo chown -R 1000:1000 /transcode_cache
```

### Step 4.5: Get User IDs

```bash
# Get your user ID and group ID
id

# Note the values:
# uid=1000(mediaserver) gid=1000(mediaserver)
# Use these for PUID and PGID in .env
```

### Step 4.6: Create .env File

```bash
nano /opt/docker/media-stack/.env
```

See `.env.example` file (created next) for all required variables.

---

## Part 5: Deploy the Stack

### Step 5.1: Review docker-compose.yml

```bash
# Review the compose file
nano /opt/docker/media-stack/docker-compose.yml

# Verify all paths and settings match your setup
```

### Step 5.2: Deploy Services in Stages

**Stage 1: VPN and Core Infrastructure**

```bash
cd /opt/docker/media-stack

# Start Gluetun first
docker compose up -d gluetun

# Wait 30 seconds for VPN to connect
sleep 30

# Check Gluetun logs
docker logs gluetun

# You should see: "Public IP address is xxx.xxx.xxx.xxx (Sweden)" or similar
# Verify it's NOT your real IP

# Test VPN from qBittorrent container
docker compose up -d qbittorrent
docker exec -it qbittorrent curl https://am.i.mullvad.net/connected
# Should return: "You are connected to Mullvad"
```

**Stage 2: Download Stack**

```bash
# Start Prowlarr and download services
docker compose up -d prowlarr autobrr flaresolverr

# Wait for services to start
sleep 10

# Check logs
docker logs prowlarr
docker logs autobrr
```

**Stage 3: Media Management (*arr)**

```bash
# Start all *arr services
docker compose up -d radarr sonarr lidarr readarr kapowarr

# Verify
docker ps | grep -E "radarr|sonarr|lidarr|readarr|kapowarr"
```

**Stage 4: Media Server**

```bash
# Start Jellyfin and related services
docker compose up -d jellyfin jellyseerr jellystat jellystat-db wizarr

# Check Jellyfin
docker logs jellyfin
```

**Stage 5: Utilities and Monitoring**

```bash
# Start remaining services
docker compose up -d tdarr bazarr cross-seed unpackerr decluttarr janitorr profilarr homarr dozzle dispatcharr

# Verify all containers are running
docker ps
```

### Step 5.3: Verify All Services

```bash
# Check that all containers are running
docker compose ps

# Should show all services as "Up" or "running"

# Check for any errors
docker compose logs --tail=50
```

---

## Part 6: Initial Configuration

### Step 6.1: qBittorrent Setup

1. Access: `http://192.168.1.100:8080`
2. Default credentials:
   - Username: `admin`
   - Password: Check logs: `docker logs qbittorrent 2>&1 | grep "temporary password"`
3. Change password immediately in **Tools → Options → Web UI**
4. Configure settings:
   - **Downloads:**
     - Default Save Path: `/data/torrents/complete`
     - Keep incomplete torrents in: `/data/torrents/incomplete`
   - **Connection:**
     - Listening Port: `8694` (must match docker-compose)
     - Enable UPnP/NAT-PMP: No
   - **BitTorrent:**
     - Enable DHT: Yes
     - Enable PeX: Yes
     - Enable Local Peer Discovery: No (VPN)
   - **Advanced:**
     - Network Interface: `tun0` (VPN interface)
     - Optional IP address to bind to: (leave empty)

5. **CRITICAL:** Test VPN leak protection:
   - Go to Tools → Execution Log
   - Look for network interface binding
   - Download a test torrent
   - If VPN disconnects, downloads should stop

### Step 6.2: Prowlarr Setup (Indexer Manager)

1. Access: `http://192.168.1.100:9696`
2. Complete setup wizard
3. Add indexers:
   - Settings → Indexers → Add Indexer
   - Add your private trackers or public ones
   - Configure FlareSolverr if needed:
     - Settings → Indexers → Add FlareSolverr Tag
     - URL: `http://localhost:8191`
4. Connect to apps:
   - Settings → Apps → Add Application
   - Add Radarr:
     - Prowlarr Server: `http://prowlarr:9696`
     - Radarr Server: `http://radarr:7878`
     - API Key: (get from Radarr → Settings → General)
   - Add Sonarr (same process)
   - Add Lidarr, Readarr, etc.

### Step 6.3: Radarr Setup (Movies)

1. Access: `http://192.168.1.100:7878`
2. Settings → Media Management:
   - Root Folder: Add `/data/media/movies`
   - File naming: Enable and configure format
3. Settings → Download Clients:
   - Add qBittorrent:
     - Host: `gluetun` (since qBit uses service networking)
     - Port: `8080`
     - Username: `admin`
     - Password: (your qBit password)
     - Category: `movies`
4. Settings → General:
   - Copy API Key (needed for Prowlarr)
5. Test: Add a movie and search for it

### Step 6.4: Sonarr Setup (TV Shows)

1. Access: `http://192.168.1.100:8989`
2. Same configuration as Radarr, but:
   - Root Folder: `/data/media/shows`
   - qBittorrent Category: `shows`
3. Settings → Profiles:
   - Review quality profiles
   - Configure to your preferences

### Step 6.5: Lidarr, Readarr, Kapowarr

Follow similar setup process:
- **Lidarr** (`http://192.168.1.100:8686`): Root folder `/data/media/music`
- **Readarr** (`http://192.168.1.100:8787`): Root folder `/data/media/books`
- **Kapowarr** (`http://192.168.1.100:5757`): Root folder `/data/media/comics`

### Step 6.6: Jellyfin Setup

1. Access: `http://192.168.1.100:8096`
2. Complete setup wizard:
   - Language: English
   - Create admin user
   - Add media libraries:
     - Movies: `/media/movies`
     - TV Shows: `/media/shows`
     - Music: `/media/music`
     - Books: `/media/books`
     - Comics: `/media/comics`
3. Configure hardware acceleration (if GPU passed through):
   - Dashboard → Playback → Transcoding
   - Hardware acceleration: Intel QuickSync (QSV) or NVIDIA NVENC
   - Enable hardware encoding: Yes
4. Create additional user accounts
5. Configure remote access settings

### Step 6.7: Jellyseerr Setup (Request Management)

1. Access: `http://192.168.1.100:5055`
2. Sign in with Jellyfin:
   - Jellyfin URL: `http://jellyfin:8096`
   - Use your Jellyfin admin credentials
3. Connect Radarr:
   - URL: `http://radarr:7878`
   - API Key: (from Radarr)
   - Root Folder: `/data/media/movies`
   - Quality Profile: HD-1080p (or your choice)
4. Connect Sonarr (same process)
5. Configure permissions for users

### Step 6.8: Additional Services

**Tdarr (Transcoding):**
- Access: `http://192.168.1.100:8265`
- Add libraries pointing to `/data/media/movies`, `/data/media/shows`, etc.
- Configure transcoding profiles (H.265, reduce bitrate, etc.)

**Bazarr (Subtitles):**
- Access: `http://192.168.1.100:6767`
- Connect to Radarr and Sonarr
- Add subtitle providers (OpenSubtitles, etc.)

**Homarr (Dashboard):**
- Access: `http://192.168.1.100:7575`
- Add all your services to dashboard
- Configure widgets

**Dozzle (Logs):**
- Access: `http://192.168.1.100:9999`
- View all container logs in real-time

---

## Part 7: Network and External Access

### Step 7.1: Port Forwarding for Seeding

For better torrent ratios, forward qBittorrent port:

1. On your router, forward port `8694` to `192.168.1.100:8694` (TCP/UDP)
2. Test port: https://www.yougetsignal.com/tools/open-ports/
3. In qBittorrent, verify the green network icon

### Step 7.2: Reverse Proxy (Optional - Advanced)

For external access with domain names (e.g., `jellyfin.yourdomain.com`):

**Option A: Use Traefik or Nginx Proxy Manager** (separate stack)
**Option B: Use Cloudflare Tunnel** (no port forwarding needed)

This is beyond the scope of this guide, but Homarr can help organize services.

---

## Verification and Testing

### Test Checklist

```bash
# 1. Verify VPN is working
docker exec qbittorrent curl https://am.i.mullvad.net/connected
# Should say "You are connected to Mullvad"

# 2. Check all containers are running
docker ps | wc -l
# Should show 30+ containers

# 3. Test Jellyfin playback
# Open Jellyfin, play a video, check for smooth playback

# 4. Test download flow
# In Radarr, add a movie → Search → Manual download
# Monitor in qBittorrent that it downloads
# Verify it appears in Jellyfin after download

# 5. Check disk usage
df -h

# 6. Check container logs for errors
docker compose logs --tail=100 | grep -i error
```

---

## Maintenance

### Daily Monitoring

```bash
# Check container status
docker ps

# Check disk space
df -h

# View logs in Dozzle
# http://192.168.1.100:9999
```

### Weekly Tasks

```bash
# Update containers
cd /opt/docker/media-stack
docker compose pull
docker compose up -d

# Clean up old images
docker image prune -a
```

### Backup Strategy

**Config Backups:**
```bash
# Backup configs weekly
tar -czf /mnt/media/backups/docker-configs-$(date +%Y%m%d).tar.gz /opt/docker

# Or use Proxmox VM backups
```

**Database Backups:**
- Jellystat creates automatic backups in `/opt/docker/media-stack/jellystat/postgres-backup`
- Backup *arr databases periodically from their config folders

---

## Troubleshooting

### VPN Not Connecting

```bash
# Check Gluetun logs
docker logs gluetun --tail=100

# Common issues:
# - Wrong private key → Regenerate on Mullvad website
# - Server offline → Change SERVER_CITIES in .env
# - MTU issues → Adjust WIREGUARD_MTU (try 1280, 1200, 1420)

# Restart Gluetun
docker compose restart gluetun
```

### Services Can't Reach VPN'd Containers

```bash
# Verify network mode
docker inspect qbittorrent | grep -i network
# Should show: "NetworkMode": "container:gluetun"

# Check Gluetun is running
docker ps | grep gluetun
```

### Permission Errors

```bash
# Fix ownership of media folders
sudo chown -R 1000:1000 /mnt/media
sudo chown -R 1000:1000 /opt/docker

# Fix permissions
sudo chmod -R 755 /mnt/media
```

### Out of Disk Space

```bash
# Clean Docker
docker system prune -a --volumes

# Clean media
# Use Decluttarr and Janitorr to auto-remove old media
```

### Jellyfin Transcoding Fails

```bash
# Verify GPU access
docker exec jellyfin ls -la /dev/dri
# Should show renderD128

# Check Jellyfin logs
docker logs jellyfin | grep -i transcode

# Verify hardware accel is enabled in Jellyfin Dashboard
```

### Can't Access Services

```bash
# Check firewall
sudo ufw status
# If enabled, allow ports:
sudo ufw allow 8096  # Jellyfin
sudo ufw allow 7575  # Homarr
# etc.

# Check if service is running
docker ps | grep <service-name>

# Check container logs
docker logs <container-name>
```

### qBittorrent VPN Leak

```bash
# Verify network binding in qBit settings
# Options → Advanced → Network Interface: tun0

# Test by stopping Gluetun
docker stop gluetun
# qBit downloads should stop immediately

# Restart
docker start gluetun
```

---

## Advanced Configuration

### Cross-Seed Setup

Edit `/opt/docker/media-stack/cross-seed/config/config.js`:
```javascript
module.exports = {
  delay: 30,
  torznab: ["http://prowlarr:9696/..."],
  outputDir: "/data/cross-seeds",
  dataDirs: ["/data/torrents/complete"],
}
```

### Janitorr Setup

Edit `/opt/docker/media-stack/janitorr/application.yml`:
```yaml
sonarr:
  url: http://sonarr:8989
  api-key: YOUR_SONARR_KEY
radarr:
  url: http://radarr:7878
  api-key: YOUR_RADARR_KEY
```

---

## Additional Resources

- **Jellyfin Docs**: https://jellyfin.org/docs/
- **Servarr Wiki**: https://wiki.servarr.com/
- **TRaSH Guides**: https://trash-guides.info/ (excellent for *arr setup)
- **Gluetun Docs**: https://github.com/qdm12/gluetun
- **r/selfhosted**: https://reddit.com/r/selfhosted
- **r/jellyfin**: https://reddit.com/r/jellyfin

---

## Credits

This stack combines best practices from multiple sources and the selfhosted community.

---

## License

This guide is provided as-is for educational purposes. Ensure you comply with your local laws regarding media downloads and VPN usage.

---

**Generated by Claude Code** - https://claude.com/claude-code
