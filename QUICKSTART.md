# Quick Start Guide - Jellyfin Media Stack

This is a condensed version for experienced users. For detailed instructions, see [README.md](README.md).

## Prerequisites Checklist

- [ ] Proxmox VE 8.0+ installed
- [ ] Ubuntu Server 24.04 LTS ISO downloaded
- [ ] Mullvad VPN account with time added
- [ ] Static IP available for the VM
- [ ] Storage for media (NFS/SMB/Local)

## 1. Create VM in Proxmox (5 minutes)

```bash
# VM Specs:
- Name: media-server
- OS: Ubuntu Server 24.04 LTS
- CPU: 4 cores (host type)
- RAM: 16GB (no ballooning)
- Disk: 100GB (SCSI, VirtIO)
- Network: VirtIO bridge

# Optional: Pass through GPU for hardware transcoding
qm set <VMID> -hostpci0 0000:00:02,rombar=0
```

## 2. Install Ubuntu (10 minutes)

1. Boot VM, install Ubuntu Server
2. Configure static IP during installation
3. Install OpenSSH
4. Update system:

```bash
sudo apt update && sudo apt upgrade -y
```

## 3. Install Docker (5 minutes)

```bash
# Add Docker repository
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

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
```

## 4. Setup Storage (5 minutes)

**For NFS:**
```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/media
echo "192.168.1.10:/volume1/media /mnt/media nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

**For SMB:**
```bash
sudo apt install -y cifs-utils
sudo mkdir -p /mnt/media
echo "username=user" | sudo tee /root/.smbcredentials
echo "password=pass" | sudo tee -a /root/.smbcredentials
sudo chmod 600 /root/.smbcredentials
echo "//192.168.1.10/media /mnt/media cifs credentials=/root/.smbcredentials,uid=1000,gid=1000 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

## 5. Get VPN Credentials (3 minutes)

1. Go to: https://mullvad.net/en/account/wireguard-config
2. Login with account number
3. Generate WireGuard key
4. Download `.conf` file
5. Extract from file:
   - `PrivateKey` → WIREGUARD_KEY
   - `Address` (IPv4) → WIREGUARD_ADD
   - Server city → VPN_SERVER_CITIES

## 6. Prepare Environment (5 minutes)

```bash
# Create directories
sudo mkdir -p /opt/docker/media-stack
cd /opt/docker/media-stack

# Clone/download this repository
git clone <repo-url> .

# Create Docker networks
docker network create proxy
docker network create jellyfin_default
docker network create starr
docker network create jellystat
docker network create gluetun_network --subnet=172.20.0.0/16

# Create config directories
mkdir -p {jellyfin,radarr,sonarr,readarr,lidarr,kapowarr,prowlarr,autobrr,jellyseerr,qbittorent,tdarr,bazarr,cross-seed,wizarr,unpackerr,dispatcharr,homarr,decluttarr,jellystat,janitorr,profilarr}/config
mkdir -p tdarr/{server,configs,logs}
mkdir -p jellystat/postgres-{data,backup}
mkdir -p wizarr/{data/database,cache}

# Set ownership
sudo chown -R 1000:1000 /opt/docker /mnt/media
```

## 7. Configure Environment Variables (5 minutes)

```bash
# Copy example env file
cp .env.example .env

# Edit .env
nano .env
```

**Minimum required variables:**
```bash
PUID=1000
PGID=1000
TZ=America/New_York
BASE_PATH=/opt/docker/media-stack
MEDIA_SHARE=/mnt/media

# VPN (from step 5)
WIREGUARD_KEY=your_private_key_here
WIREGUARD_ADD=10.x.x.x/32
VPN_SERVER_CITIES=Amsterdam

# Passwords (generate strong ones)
QBIT_PASS=your_password
JELLYSTATDB_PASS=your_password
JWT_SECRET=$(openssl rand -base64 32)
HOMARR_KEY=$(openssl rand -base64 32)

# URLs
JELLYFIN_URL=http://192.168.1.100:8096
WIZARR_URL=http://192.168.1.100:5690

# Network
RADARR_IPV4=172.20.0.10
SONARR_IPV4=172.20.0.11

# API Keys (leave empty, fill after first start)
SONARR_KEY=
RADARR_KEY=
LIDARR_KEY=
READARR_KEY=
```

## 8. Deploy Stack (10 minutes)

```bash
cd /opt/docker/media-stack

# Start VPN first
docker compose up -d gluetun
sleep 30

# Verify VPN
docker logs gluetun | grep -i "ip"

# Start download stack
docker compose up -d qbittorrent prowlarr autobrr flaresolverr
sleep 10

# Start *arr services
docker compose up -d radarr sonarr lidarr readarr kapowarr
sleep 10

# Start media server
docker compose up -d jellyfin jellyseerr jellystat jellystat-db wizarr
sleep 10

# Start utilities
docker compose up -d tdarr bazarr cross-seed unpackerr decluttarr janitorr profilarr homarr dozzle dispatcharr

# Verify all running
docker ps
```

## 9. Initial Configuration (30 minutes)

### Get qBittorrent Password
```bash
docker logs qbittorrent 2>&1 | grep "temporary password"
```

### Configure Services

| Service | URL | Purpose | Priority |
|---------|-----|---------|----------|
| qBittorrent | http://IP:8080 | Torrent client | HIGH |
| Prowlarr | http://IP:9696 | Indexer manager | HIGH |
| Radarr | http://IP:7878 | Movies | HIGH |
| Sonarr | http://IP:8989 | TV Shows | HIGH |
| Jellyfin | http://IP:8096 | Media server | HIGH |
| Jellyseerr | http://IP:5055 | Request system | MEDIUM |
| Homarr | http://IP:7575 | Dashboard | MEDIUM |
| Lidarr | http://IP:8686 | Music | LOW |
| Readarr | http://IP:8787 | Books | LOW |
| Kapowarr | http://IP:5757 | Comics | LOW |
| Bazarr | http://IP:6767 | Subtitles | LOW |
| Tdarr | http://IP:8265 | Transcoding | LOW |
| Dozzle | http://IP:9999 | Log viewer | UTILITY |

### Quick Configuration Steps

**1. qBittorrent:**
- Login with admin + temporary password
- Change password immediately
- Set Downloads → Default Save Path: `/data/torrents/complete`
- Set Connection → Listening Port: `8694`
- Set Advanced → Network Interface: `tun0`

**2. Prowlarr:**
- Add indexers
- Connect apps (Radarr, Sonarr, etc.) using their API keys

**3. Radarr:**
- Settings → Media Management → Root Folder: `/data/media/movies`
- Settings → Download Clients → Add qBittorrent (host: `gluetun`, port: `8080`)
- Copy API Key from Settings → General

**4. Sonarr:**
- Same as Radarr, but root folder: `/data/media/shows`

**5. Jellyfin:**
- Complete setup wizard
- Add libraries: `/media/movies`, `/media/shows`, etc.
- Enable hardware transcoding (if GPU available)

**6. Jellyseerr:**
- Connect to Jellyfin (URL: `http://jellyfin:8096`)
- Connect Radarr and Sonarr with API keys

## 10. Get API Keys and Update .env (5 minutes)

```bash
# Get API keys from each service (Settings → General → API Key)
# Then update .env file

nano /opt/docker/media-stack/.env

# Update these lines:
SONARR_KEY=<copy from Sonarr>
RADARR_KEY=<copy from Radarr>
LIDARR_KEY=<copy from Lidarr>
READARR_KEY=<copy from Readarr>

# Restart affected services
docker compose up -d unpackerr decluttarr
```

## 11. Test Everything (10 minutes)

```bash
# Test VPN
docker exec qbittorrent curl https://am.i.mullvad.net/connected
# Should say: "You are connected to Mullvad"

# Test download flow
# 1. Go to Radarr → Add Movie → Search
# 2. Pick a movie, manual search, download
# 3. Watch in qBittorrent (should download)
# 4. After completion, should appear in Jellyfin

# Port forward qBittorrent for better seeding
# On your router: Forward port 8694 (TCP/UDP) to VM IP
```

## 12. Optional: Create Folder Structure (5 minutes)

```bash
# Create media folders if they don't exist
mkdir -p /mnt/media/media/{movies,shows,music,books,comics}
mkdir -p /mnt/media/torrents/{complete,incomplete,cross-seed,watch}
mkdir -p /mnt/media/temp_downloads

# Set permissions
sudo chown -R 1000:1000 /mnt/media
sudo chmod -R 755 /mnt/media
```

## Maintenance Commands

```bash
# View all logs
docker compose logs -f

# View specific service
docker logs -f jellyfin

# Restart service
docker compose restart jellyfin

# Update all containers
cd /opt/docker/media-stack
docker compose pull
docker compose up -d

# Clean up
docker system prune -a

# Backup configs
tar -czf backup-$(date +%Y%m%d).tar.gz /opt/docker/media-stack
```

## Common Issues

**VPN not connecting:**
```bash
docker logs gluetun
# Check WIREGUARD_KEY and WIREGUARD_ADD are correct
```

**Can't access qBittorrent:**
```bash
# It's behind Gluetun, access via Gluetun's ports
# URL: http://VM_IP:8080
```

**Permission errors:**
```bash
sudo chown -R 1000:1000 /mnt/media /opt/docker
sudo chmod -R 755 /mnt/media
```

**Services won't start:**
```bash
# Check networks exist
docker network ls

# Recreate if needed
docker network create proxy
docker network create starr
# etc.
```

## What's Next?

1. **Configure TRaSH Guides** for optimal quality settings
   - Visit: https://trash-guides.info/
   - Apply custom formats in Radarr/Sonarr

2. **Setup Reverse Proxy** for external access
   - Traefik, Nginx Proxy Manager, or Cloudflare Tunnel

3. **Add More Indexers** in Prowlarr
   - Private trackers for better content

4. **Configure Tdarr** for space savings
   - Transcode to H.265/HEVC

5. **Setup Autobrr** for better ratios
   - Connect to IRC channels of your trackers

6. **Customize Homarr Dashboard**
   - Add all services with beautiful widgets

## Support

- **Documentation**: See full [README.md](README.md)
- **Logs**: http://VM_IP:9999 (Dozzle)
- **Issues**: Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)

---

**Total Setup Time: ~90 minutes**

---

Generated with Claude Code - https://claude.com/claude-code
