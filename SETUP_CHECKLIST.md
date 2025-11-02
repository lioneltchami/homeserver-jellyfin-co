# Setup Checklist - Jellyfin Media Stack

Use this checklist to track your progress during setup. Check off items as you complete them.

---

## Pre-Setup Preparation

### Hardware & Accounts
- [ ] Proxmox VE 8.0+ installed and accessible
- [ ] Mullvad VPN account created
- [ ] Mullvad account has time added (minimum 5 EUR)
- [ ] Ubuntu Server 24.04 LTS ISO downloaded
- [ ] Storage solution decided (NFS/SMB/Local)
- [ ] Static IP address allocated for VM (e.g., 192.168.1.100)

### Planning
- [ ] Decided on CPU allocation (4+ cores recommended)
- [ ] Decided on RAM allocation (16GB minimum, 32GB recommended)
- [ ] Decided on storage allocation (100GB+ for configs)
- [ ] Media storage location identified (NAS path or local disks)
- [ ] Reviewed services list and understand what each does

---

## Part 1: Proxmox VM Setup

### Create VM
- [ ] Logged into Proxmox web interface
- [ ] Clicked "Create VM"
- [ ] Set VM ID and name (e.g., 100, media-server)
- [ ] Selected Ubuntu Server 24.04 LTS ISO
- [ ] Configured System (UEFI, q35, VirtIO SCSI)
- [ ] Configured Disk (100GB, SSD emulation if applicable)
- [ ] Configured CPU (4+ cores, host type)
- [ ] Configured Memory (16GB, no ballooning)
- [ ] Configured Network (VirtIO, bridge vmbr0)
- [ ] VM created successfully

### GPU Passthrough (Optional)
- [ ] Enabled IOMMU in Proxmox (if doing GPU passthrough)
- [ ] Rebooted Proxmox host (if IOMMU enabled)
- [ ] Passed through GPU to VM (`qm set` command)
- [ ] Verified GPU shows in VM hardware tab

### Install Ubuntu
- [ ] Started VM
- [ ] Opened VM console
- [ ] Selected language (English)
- [ ] Configured keyboard layout
- [ ] Configured network with static IP
  - IP: ________________
  - Gateway: ________________
  - DNS: 1.1.1.1, 8.8.8.8
- [ ] Used entire disk for installation
- [ ] Created user account
  - Username: ________________
  - Password: (saved securely)
- [ ] Installed OpenSSH server
- [ ] Completed installation
- [ ] Rebooted VM
- [ ] Successfully logged in via SSH

### Update Ubuntu
- [ ] Ran `sudo apt update`
- [ ] Ran `sudo apt upgrade -y`
- [ ] Installed prerequisites (curl, wget, git, etc.)
- [ ] Installed GPU drivers (if using hardware transcoding)
- [ ] Added user to video/render groups (if using GPU)
- [ ] Verified GPU access with `ls -la /dev/dri` (if applicable)

### Configure Storage
- [ ] Installed NFS client (if using NFS)
- [ ] Installed CIFS tools (if using SMB)
- [ ] Created mount point: `/mnt/media`
- [ ] Tested manual mount
- [ ] Added to `/etc/fstab` for persistence
- [ ] Verified mount survives reboot
- [ ] Verified write permissions to mount

---

## Part 2: Docker Installation

- [ ] Removed old Docker versions (if any)
- [ ] Added Docker GPG key
- [ ] Added Docker repository
- [ ] Installed Docker CE and Docker Compose
- [ ] Verified installation: `docker --version`
- [ ] Verified installation: `docker compose version`
- [ ] Added user to docker group
- [ ] Logged out and back in (or ran `newgrp docker`)
- [ ] Tested Docker: `docker run hello-world`
- [ ] Created Docker daemon config (`/etc/docker/daemon.json`)
- [ ] Restarted Docker service
- [ ] Enabled Docker on boot

---

## Part 3: VPN Configuration

### Mullvad Account
- [ ] Logged into Mullvad account
- [ ] Went to WireGuard config page
- [ ] Generated new WireGuard key
- [ ] Selected server location (noted: ________________)
- [ ] Downloaded `.conf` file

### Extract Credentials
- [ ] Opened `.conf` file
- [ ] Copied `PrivateKey` value
  - Saved as: WIREGUARD_KEY
- [ ] Copied `Address` IPv4 value (e.g., 10.x.x.x/32)
  - Saved as: WIREGUARD_ADD
- [ ] Noted server city name
  - Saved as: VPN_SERVER_CITIES

---

## Part 4: Environment Preparation

### Create Directories
- [ ] Created `/opt/docker/media-stack`
- [ ] Changed to project directory
- [ ] Set ownership: `sudo chown -R $USER:$USER /opt/docker`

### Download Project
- [ ] Cloned/downloaded this project to `/opt/docker/media-stack`
- [ ] Verified files present: `docker-compose.yml`, `.env.example`

### Create Docker Networks
- [ ] Created `proxy` network
- [ ] Created `jellyfin_default` network
- [ ] Created `starr` network
- [ ] Created `jellystat` network
- [ ] Created `gluetun_network` with subnet
- [ ] Verified all networks: `docker network ls`

### Create Config Directories
- [ ] Created all service config directories
- [ ] Created Tdarr subdirectories
- [ ] Created Jellystat subdirectories
- [ ] Created Wizarr subdirectories
- [ ] Created transcode cache directory (optional)
- [ ] Set ownership: `sudo chown -R 1000:1000 /opt/docker /mnt/media`

### Get User IDs
- [ ] Ran `id` command
- [ ] Noted PUID: ________________
- [ ] Noted PGID: ________________

### Configure .env File
- [ ] Copied `.env.example` to `.env`
- [ ] Set PUID and PGID
- [ ] Set TZ (timezone)
- [ ] Set BASE_PATH
- [ ] Set MEDIA_SHARE
- [ ] Set WIREGUARD_KEY (from Mullvad)
- [ ] Set WIREGUARD_ADD (from Mullvad)
- [ ] Set VPN_SERVER_CITIES
- [ ] Set WIREGUARD_MTU (default 1200)
- [ ] Set JELLYFIN_URL
- [ ] Set WIZARR_URL
- [ ] Set QBIT_USER
- [ ] Set QBIT_PASS (strong password)
- [ ] Set JELLYSTATDB_USER
- [ ] Set JELLYSTATDB_PASS (strong password)
- [ ] Generated JWT_SECRET: `openssl rand -base64 32`
- [ ] Generated HOMARR_KEY: `openssl rand -base64 32`
- [ ] Set RADARR_IPV4 and SONARR_IPV4
- [ ] Left API keys empty (will fill after first start)

---

## Part 5: Deploy the Stack

### Stage 1: VPN
- [ ] Started Gluetun: `docker compose up -d gluetun`
- [ ] Waited 30 seconds
- [ ] Checked Gluetun logs for successful connection
- [ ] Verified VPN IP is not your real IP
- [ ] Started qBittorrent: `docker compose up -d qbittorrent`
- [ ] Tested VPN: `docker exec -it qbittorrent curl https://am.i.mullvad.net/connected`
- [ ] Confirmed: "You are connected to Mullvad"

### Stage 2: Download Stack
- [ ] Started Prowlarr
- [ ] Started Autobrr
- [ ] Started FlareSolverr
- [ ] Verified all started successfully

### Stage 3: *arr Services
- [ ] Started Radarr
- [ ] Started Sonarr
- [ ] Started Lidarr
- [ ] Started Readarr
- [ ] Started Kapowarr
- [ ] Verified all running: `docker ps`

### Stage 4: Media Server
- [ ] Started Jellyfin
- [ ] Started Jellyseerr
- [ ] Started Jellystat database
- [ ] Started Jellystat
- [ ] Started Wizarr
- [ ] Verified all running

### Stage 5: Utilities
- [ ] Started Tdarr
- [ ] Started Bazarr
- [ ] Started Cross-seed
- [ ] Started Unpackerr
- [ ] Started Decluttarr
- [ ] Started Janitorr
- [ ] Started Profilarr
- [ ] Started Homarr
- [ ] Started Dozzle
- [ ] Started Dispatcharr
- [ ] Verified ALL containers running: `docker compose ps`

---

## Part 6: Initial Configuration

### qBittorrent
- [ ] Accessed WebUI at http://VM_IP:8080
- [ ] Got temporary password from logs
- [ ] Logged in successfully
- [ ] Changed admin password immediately
- [ ] Set Default Save Path: `/data/torrents/complete`
- [ ] Set Incomplete torrents path: `/data/torrents/incomplete`
- [ ] Set Listening Port: 8694
- [ ] Disabled UPnP/NAT-PMP
- [ ] Enabled DHT
- [ ] Enabled PeX
- [ ] Set Network Interface: `tun0`
- [ ] Saved settings
- [ ] Tested VPN kill switch (stopped Gluetun, downloads should stop)

### Prowlarr
- [ ] Accessed at http://VM_IP:9696
- [ ] Completed setup wizard
- [ ] Added indexers (at least 2-3)
- [ ] Configured FlareSolverr if needed (http://localhost:8191)
- [ ] Connected to Radarr (Settings → Apps)
- [ ] Connected to Sonarr
- [ ] Connected to Lidarr (optional)
- [ ] Connected to Readarr (optional)
- [ ] Tested indexer search

### Radarr
- [ ] Accessed at http://VM_IP:7878
- [ ] Added Root Folder: `/data/media/movies`
- [ ] Configured file naming
- [ ] Added qBittorrent download client
  - Host: `gluetun`
  - Port: 8080
  - Category: `movies`
- [ ] Tested connection to qBittorrent
- [ ] Copied API Key (Settings → General → Security)
- [ ] Updated Prowlarr connection with API key

### Sonarr
- [ ] Accessed at http://VM_IP:8989
- [ ] Added Root Folder: `/data/media/shows`
- [ ] Configured file naming
- [ ] Added qBittorrent download client
  - Host: `gluetun`
  - Category: `shows`
- [ ] Tested connection
- [ ] Copied API Key
- [ ] Updated Prowlarr connection

### Lidarr (Optional)
- [ ] Accessed at http://VM_IP:8686
- [ ] Added Root Folder: `/data/media/music`
- [ ] Configured download client
- [ ] Copied API Key

### Readarr (Optional)
- [ ] Accessed at http://VM_IP:8787
- [ ] Added Root Folder: `/data/media/books`
- [ ] Configured download client
- [ ] Copied API Key

### Kapowarr (Optional)
- [ ] Accessed at http://VM_IP:5757 (via Gluetun)
- [ ] Added Root Folder: `/data/media/comics`
- [ ] Configured settings

### Jellyfin
- [ ] Accessed at http://VM_IP:8096
- [ ] Selected language
- [ ] Created admin user
  - Username: ________________
  - Password: (saved securely)
- [ ] Added Movies library: `/media/movies`
- [ ] Added TV Shows library: `/media/shows`
- [ ] Added Music library: `/media/music` (optional)
- [ ] Added Books library: `/media/books` (optional)
- [ ] Added Comics library: `/media/comics` (optional)
- [ ] Configured hardware acceleration (if GPU present)
  - Dashboard → Playback → Transcoding
  - Selected: Intel QuickSync or NVIDIA NVENC
- [ ] Enabled hardware encoding
- [ ] Tested playback with a sample file

### Jellyseerr
- [ ] Accessed at http://VM_IP:5055
- [ ] Signed in with Jellyfin
  - Jellyfin URL: `http://jellyfin:8096`
- [ ] Connected Radarr
  - URL: `http://radarr:7878`
  - API Key: (from Radarr)
  - Root Folder: `/data/media/movies`
- [ ] Connected Sonarr
  - URL: `http://sonarr:8989`
  - API Key: (from Sonarr)
  - Root Folder: `/data/media/shows`
- [ ] Configured user permissions
- [ ] Tested making a request

### Tdarr (Optional)
- [ ] Accessed at http://VM_IP:8265
- [ ] Added libraries
- [ ] Configured transcoding profiles
- [ ] Enabled hardware acceleration (if available)

### Bazarr (Optional)
- [ ] Accessed at http://VM_IP:6767
- [ ] Connected to Radarr
- [ ] Connected to Sonarr
- [ ] Added subtitle providers
- [ ] Configured languages

### Homarr
- [ ] Accessed at http://VM_IP:7575
- [ ] Added all services to dashboard
- [ ] Configured layout
- [ ] Added widgets

### Dozzle
- [ ] Accessed at http://VM_IP:9999
- [ ] Verified can see all container logs
- [ ] Bookmarked for troubleshooting

---

## Part 7: Update API Keys in .env

- [ ] Got Sonarr API key from Settings → General
- [ ] Got Radarr API key
- [ ] Got Lidarr API key (if using)
- [ ] Got Readarr API key (if using)
- [ ] Updated `.env` file with all API keys
- [ ] Restarted Unpackerr: `docker compose restart unpackerr`
- [ ] Restarted Decluttarr: `docker compose restart decluttarr`

---

## Part 8: Network & External Access

### Port Forwarding (For better seeding)
- [ ] Logged into router
- [ ] Forwarded port 8694 TCP/UDP to VM IP
- [ ] Tested port at https://www.yougetsignal.com/tools/open-ports/
- [ ] Verified green network icon in qBittorrent

### Reverse Proxy (Optional - Advanced)
- [ ] Decided on reverse proxy solution (Traefik/NPM/Cloudflare)
- [ ] Set up reverse proxy (if doing)
- [ ] Configured domain names
- [ ] Configured SSL certificates
- [ ] Tested external access

---

## Part 9: Testing & Verification

### VPN Tests
- [ ] VPN connection test: `docker logs gluetun | grep -i ip`
- [ ] VPN leak test: `docker exec qbittorrent curl https://am.i.mullvad.net/connected`
- [ ] Kill switch test: Stopped Gluetun, downloads stopped
- [ ] Restarted Gluetun, downloads resumed

### Download Flow Test
- [ ] Opened Radarr
- [ ] Added a movie
- [ ] Searched for releases
- [ ] Manually downloaded a release
- [ ] Monitored download in qBittorrent
- [ ] Waited for completion
- [ ] Verified Radarr imported it
- [ ] Verified it appeared in Jellyfin
- [ ] Played in Jellyfin successfully

### Request Flow Test (Jellyseerr)
- [ ] Requested a movie in Jellyseerr
- [ ] Verified it appeared in Radarr
- [ ] Verified it started downloading
- [ ] Completed and imported successfully

### Container Health Check
- [ ] All containers showing "Up": `docker compose ps`
- [ ] No errors in logs: `docker compose logs --tail=100 | grep -i error`
- [ ] Disk space healthy: `df -h`

### Performance Tests
- [ ] Jellyfin playback smooth
- [ ] Hardware transcoding working (if enabled)
- [ ] Download speeds acceptable
- [ ] CPU usage reasonable: `docker stats`
- [ ] Memory usage reasonable

---

## Part 10: Post-Setup

### Media Organization
- [ ] Created media folder structure
  - `/mnt/media/media/movies/`
  - `/mnt/media/media/shows/`
  - `/mnt/media/media/music/`
  - `/mnt/media/torrents/complete/`
  - `/mnt/media/torrents/incomplete/`
- [ ] Set correct permissions
- [ ] Added some test content

### Backups
- [ ] Created backup directory: `/mnt/media/backups/`
- [ ] Created backup script
- [ ] Tested backup script
- [ ] Scheduled automatic backups (cron)
- [ ] Documented restore procedure

### Documentation
- [ ] Documented VM IP address: ________________
- [ ] Documented all usernames and passwords (in password manager)
- [ ] Documented VPN credentials
- [ ] Bookmarked all service URLs
- [ ] Saved copy of `.env` file securely
- [ ] Created network diagram (optional)

### User Accounts
- [ ] Created Jellyfin accounts for family/friends
- [ ] Set up Wizarr invite links (if using)
- [ ] Configured user permissions in Jellyseerr
- [ ] Tested user access

### Monitoring
- [ ] Set up Jellystat for statistics
- [ ] Configured Homarr dashboard
- [ ] Set up alerts (optional)
- [ ] Documented where to check logs (Dozzle)

---

## Part 11: Advanced Configuration (Optional)

### TRaSH Guides
- [ ] Visited https://trash-guides.info/
- [ ] Applied Radarr quality profiles
- [ ] Applied Sonarr quality profiles
- [ ] Applied custom formats
- [ ] Configured naming schemes

### Cross-Seed
- [ ] Configured `/opt/docker/media-stack/cross-seed/config/config.js`
- [ ] Added Prowlarr Torznab feeds
- [ ] Tested cross-seeding

### Janitorr
- [ ] Configured `/opt/docker/media-stack/janitorr/application.yml`
- [ ] Set cleanup rules
- [ ] Tested cleanup

### Autobrr
- [ ] Connected to IRC channels
- [ ] Configured filters
- [ ] Tested auto-grabbing

### Tdarr
- [ ] Created transcoding rules
- [ ] Set H.265 conversion
- [ ] Limited concurrent jobs
- [ ] Started transcoding queue

---

## Part 12: Maintenance Setup

### Update Schedule
- [ ] Scheduled monthly updates
- [ ] Created update script
- [ ] Tested update procedure
- [ ] Documented rollback procedure

### Monitoring Schedule
- [ ] Weekly: Check disk space
- [ ] Weekly: Check container health
- [ ] Monthly: Review logs for errors
- [ ] Monthly: Update containers
- [ ] Quarterly: Test backups

### Security
- [ ] Changed all default passwords
- [ ] Enabled 2FA where possible
- [ ] Reviewed firewall rules
- [ ] Set up fail2ban (optional)
- [ ] Configured VPN leak alerts

---

## Final Checklist

- [ ] All containers running
- [ ] VPN working and leak-proof
- [ ] Can download and play media
- [ ] All services accessible
- [ ] Backups configured
- [ ] Documentation complete
- [ ] Users can access and request media
- [ ] Monitoring in place
- [ ] Update schedule set
- [ ] Read through TROUBLESHOOTING.md
- [ ] Bookmarked important resources

---

## Success Criteria

✅ **Setup is complete when:**
1. You can request a movie in Jellyseerr
2. It automatically downloads via qBittorrent (through VPN)
3. It imports into Radarr
4. It appears in Jellyfin
5. You can play it smoothly
6. All services are accessible via web browser
7. VPN leak test passes
8. Backups are working

---

## Notes & Customizations

Use this space to note any customizations or deviations from the standard setup:

```
[Your notes here]
```

---

## Completion

**Setup started:** ________________

**Setup completed:** ________________

**Total time:** ________________

**Completed by:** ________________

---

Generated with Claude Code - https://claude.com/claude-code
