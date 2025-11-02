# Troubleshooting Guide - Jellyfin Media Stack

## Table of Contents
- [General Diagnostics](#general-diagnostics)
- [VPN Issues](#vpn-issues)
- [Network Issues](#network-issues)
- [Container Issues](#container-issues)
- [Permission Issues](#permission-issues)
- [Storage Issues](#storage-issues)
- [Service-Specific Issues](#service-specific-issues)
- [Performance Issues](#performance-issues)
- [Update Issues](#update-issues)

---

## General Diagnostics

### Check All Containers Status
```bash
cd /opt/docker/media-stack
docker compose ps

# Look for:
# - "Up" status for all services
# - "Restarting" indicates a problem
# - "Exited" means the container crashed
```

### View Container Logs
```bash
# All services
docker compose logs --tail=100

# Specific service
docker logs jellyfin --tail=100

# Follow logs in real-time
docker logs -f gluetun

# Or use Dozzle web interface
# http://YOUR_VM_IP:9999
```

### Check System Resources
```bash
# Disk space
df -h

# Memory usage
free -h

# Docker disk usage
docker system df

# CPU and memory per container
docker stats
```

### Restart Services
```bash
# Single service
docker compose restart jellyfin

# Multiple services
docker compose restart radarr sonarr

# All services
docker compose down
docker compose up -d
```

---

## VPN Issues

### Gluetun Won't Connect

**Symptoms:**
- Containers using VPN can't access internet
- Gluetun logs show connection errors
- qBittorrent not accessible

**Check Gluetun logs:**
```bash
docker logs gluetun --tail=100

# Look for errors like:
# - "authentication failed"
# - "cannot resolve host"
# - "timeout"
```

**Solutions:**

1. **Verify VPN Credentials**
   ```bash
   # Check .env file
   cat /opt/docker/media-stack/.env | grep WIREGUARD

   # Should show:
   # WIREGUARD_KEY=your_private_key
   # WIREGUARD_ADD=10.x.x.x/32
   ```

2. **Regenerate WireGuard Config**
   - Go to: https://mullvad.net/en/account/wireguard-config
   - Delete old key, generate new one
   - Download new .conf file
   - Update WIREGUARD_KEY and WIREGUARD_ADD in .env
   - Restart Gluetun:
     ```bash
     docker compose restart gluetun
     ```

3. **Change VPN Server**
   ```bash
   # Edit .env
   nano /opt/docker/media-stack/.env

   # Change VPN_SERVER_CITIES to different city
   # Examples: Stockholm, New York, Tokyo

   # Restart
   docker compose restart gluetun
   ```

4. **Adjust MTU**
   ```bash
   # Edit .env
   nano /opt/docker/media-stack/.env

   # Try different values:
   WIREGUARD_MTU=1280
   # or
   WIREGUARD_MTU=1420
   # or
   WIREGUARD_MTU=1380

   # Restart
   docker compose restart gluetun
   ```

5. **Update Mullvad Servers**
   ```bash
   # Edit docker-compose.yml and uncomment update command
   docker compose down gluetun
   docker compose run gluetun update -enduser -providers mullvad
   docker compose up -d gluetun
   ```

### VPN Connected But No Internet

**Test VPN connection:**
```bash
docker exec qbittorrent curl https://am.i.mullvad.net/connected
# Should return: "You are connected to Mullvad"

# If not working, check:
docker exec qbittorrent curl https://ipinfo.io/ip
# Should NOT be your real IP
```

**Solutions:**

1. **Check DNS**
   ```bash
   docker exec qbittorrent nslookup google.com

   # If fails, Gluetun might have DNS issues
   docker compose restart gluetun
   ```

2. **Verify Network Mode**
   ```bash
   docker inspect qbittorrent | grep -i networkmode
   # Should show: "NetworkMode": "container:<gluetun_id>"
   ```

### VPN Leaking (Downloads Continue When VPN Disconnected)

**This is CRITICAL for privacy!**

**Test:**
```bash
# Stop Gluetun
docker stop gluetun

# Try to download in qBittorrent
# Downloads should STOP immediately

# Restart Gluetun
docker start gluetun
```

**If downloads continue when VPN is down:**

1. **Check qBittorrent Network Binding**
   - Go to qBittorrent WebUI → Options → Advanced
   - Network Interface: Must be set to `tun0`
   - Optional IP address: Leave empty

2. **Verify network_mode in docker-compose.yml**
   ```yaml
   qbittorrent:
     network_mode: "service:gluetun"
   ```

3. **Restart containers in correct order**
   ```bash
   docker compose down
   docker compose up -d gluetun
   sleep 30
   docker compose up -d qbittorrent prowlarr autobrr
   ```

---

## Network Issues

### Cannot Access Services via Browser

**Symptoms:**
- Can't access Jellyfin at http://VM_IP:8096
- Browser shows "Connection refused"

**Solutions:**

1. **Verify Container is Running**
   ```bash
   docker ps | grep jellyfin
   # Should show "Up" status
   ```

2. **Check Port Binding**
   ```bash
   docker port jellyfin
   # Should show: 8096/tcp -> 0.0.0.0:8096
   ```

3. **Check Firewall**
   ```bash
   # Ubuntu firewall
   sudo ufw status

   # If active, allow port
   sudo ufw allow 8096

   # Or disable for testing (not recommended for production)
   sudo ufw disable
   ```

4. **Test from VM itself**
   ```bash
   # From VM
   curl http://localhost:8096

   # If works locally but not remotely, it's a network issue
   ```

5. **Check Proxmox Firewall**
   - In Proxmox web UI
   - Select VM → Firewall
   - Ensure ports are allowed

### External Networks Not Created

**Error:**
```
ERROR: Network proxy declared as external, but could not be found
```

**Solution:**
```bash
# Create all required networks
docker network create proxy
docker network create jellyfin_default
docker network create starr
docker network create jellystat
docker network create gluetun_network --subnet=172.20.0.0/16

# Verify
docker network ls
```

### Services Can't Communicate

**Symptoms:**
- Prowlarr can't connect to Radarr
- Radarr can't connect to qBittorrent
- Error: "Connection refused" or "Host not found"

**Solutions:**

1. **Check if services are on same network**
   ```bash
   docker inspect radarr | grep -A 20 Networks
   docker inspect sonarr | grep -A 20 Networks
   ```

2. **Use container names, not localhost**
   - ✅ Correct: `http://radarr:7878`
   - ❌ Wrong: `http://localhost:7878`
   - ❌ Wrong: `http://127.0.0.1:7878`

3. **For VPN'd services, use gluetun**
   - qBittorrent URL in Radarr: `http://gluetun:8080`
   - Prowlarr URL from outside: `http://gluetun:9696`

4. **Restart both services**
   ```bash
   docker compose restart radarr prowlarr
   ```

---

## Container Issues

### Container Keeps Restarting

**Check logs:**
```bash
docker logs <container_name> --tail=100

# Common errors:
# - Permission denied
# - Cannot bind to port
# - Configuration error
```

**Solutions:**

1. **Permission Issues**
   ```bash
   sudo chown -R 1000:1000 /opt/docker/media-stack
   sudo chown -R 1000:1000 /mnt/media
   ```

2. **Port Already in Use**
   ```bash
   # Check what's using the port
   sudo netstat -tulpn | grep :8096

   # Kill the process or change port in docker-compose.yml
   ```

3. **Configuration Error**
   ```bash
   # Remove config and start fresh
   sudo rm -rf /opt/docker/media-stack/jellyfin/config/*
   docker compose restart jellyfin
   ```

### Container Exited with Error

```bash
# Get exit code
docker inspect <container> --format='{{.State.ExitCode}}'

# Common exit codes:
# 0 = Success (intentional stop)
# 1 = General error
# 137 = Out of memory (killed)
# 139 = Segmentation fault
```

**For exit code 137 (OOM):**
```bash
# Check memory
free -h

# Add more RAM to VM in Proxmox
# Or limit memory usage in docker-compose.yml:
services:
  jellyfin:
    mem_limit: 2g
```

### Cannot Remove Container

```bash
# Force remove
docker rm -f <container_name>

# If still stuck
docker compose down --remove-orphans

# Nuclear option (removes everything)
docker compose down -v
docker compose up -d
```

---

## Permission Issues

### Permission Denied Errors

**Symptoms:**
- Logs show "Permission denied"
- Can't write to media folders
- Configuration won't save

**Check current ownership:**
```bash
ls -la /mnt/media
ls -la /opt/docker/media-stack
```

**Fix ownership:**
```bash
# Get your PUID/PGID
id

# Set ownership (use your PUID/PGID from .env)
sudo chown -R 1000:1000 /mnt/media
sudo chown -R 1000:1000 /opt/docker/media-stack

# Set permissions
sudo chmod -R 755 /mnt/media
sudo chmod -R 755 /opt/docker/media-stack
```

### GPU Permission Denied (Transcoding)

**Symptoms:**
- Jellyfin/Tdarr can't access /dev/dri
- Hardware transcoding fails

**Solution:**
```bash
# On VM, check GPU
ls -la /dev/dri
# Should show renderD128 and card0

# Add user to groups
sudo usermod -aG video,render $USER

# Check groups
groups

# Set permissions
sudo chmod -R 777 /dev/dri

# Restart containers
docker compose restart jellyfin tdarr
```

---

## Storage Issues

### Out of Disk Space

**Check disk usage:**
```bash
df -h

# Docker disk usage
docker system df

# Find large files
du -sh /opt/docker/media-stack/* | sort -h
du -sh /mnt/media/* | sort -h
```

**Clean up:**
```bash
# Remove unused Docker data
docker system prune -a --volumes
# WARNING: This removes stopped containers, unused networks, images, and volumes

# Clean logs
sudo truncate -s 0 /opt/docker/media-stack/*/config/logs/*.log

# Use Decluttarr/Janitorr to remove old media
# Configure in their respective configs
```

### Media Share Not Mounted

**Symptoms:**
- /mnt/media is empty
- Jellyfin shows "No media found"

**Check mount:**
```bash
df -h | grep media
mount | grep media
```

**Fix:**

1. **For NFS:**
   ```bash
   sudo mount -a

   # If fails, check /etc/fstab
   cat /etc/fstab | grep media

   # Test manual mount
   sudo mount -t nfs 192.168.1.10:/volume1/media /mnt/media
   ```

2. **For SMB:**
   ```bash
   sudo mount -a

   # Check credentials
   cat /root/.smbcredentials

   # Test manual mount
   sudo mount -t cifs //192.168.1.10/media /mnt/media -o credentials=/root/.smbcredentials
   ```

3. **Make mount persistent:**
   ```bash
   # Add to /etc/fstab
   sudo nano /etc/fstab

   # For NFS:
   192.168.1.10:/volume1/media /mnt/media nfs defaults,_netdev 0 0

   # For SMB:
   //192.168.1.10/media /mnt/media cifs credentials=/root/.smbcredentials,uid=1000,gid=1000 0 0
   ```

---

## Service-Specific Issues

### Jellyfin

**Can't play videos / Transcoding fails:**
```bash
# Check logs
docker logs jellyfin | grep -i transcode

# Verify GPU passthrough
docker exec jellyfin ls -la /dev/dri

# Check Jellyfin Dashboard → Playback → Transcoding
# Enable: Hardware acceleration
# Select: Intel QuickSync (QSV) or NVIDIA NVENC
```

**Libraries not scanning:**
- Dashboard → Scheduled Tasks → Scan Media Library → Run now
- Check paths: `/media/movies`, `/media/shows` (not `/mnt/media/...`)

### qBittorrent

**Can't login / Forgot password:**
```bash
# Get temporary password from logs
docker logs qbittorrent 2>&1 | grep "temporary password"

# Or reset config
docker compose down qbittorrent
sudo rm -rf /opt/docker/media-stack/qbittorent/config/qBittorrent/config/qBittorrent.conf
docker compose up -d qbittorrent
```

**Downloads not starting:**
- Check VPN is connected (see VPN Issues section)
- Verify disk space
- Check logs: `docker logs qbittorrent`

**Poor seeding / No incoming connections:**
- Port forward 8694 on your router to VM IP
- Verify with: https://www.yougetsignal.com/tools/open-ports/
- In qBit, check for green network icon

### Prowlarr

**Can't add indexers:**
- Check if FlareSolverr is needed: `docker logs flaresolverr`
- Add FlareSolverr tag to indexer in Prowlarr
- FlareSolverr URL: `http://localhost:8191`

**Indexers failing:**
- Update indexers: Settings → Indexers → Update All
- Some may be down temporarily, check indexer status websites

### Radarr/Sonarr

**Movies/Shows not downloading:**
1. Check if indexers are working in Prowlarr
2. Verify download client (qBittorrent) is connected
3. Check quality profile allows the release
4. Check logs: Activity → Queue

**Import failing:**
- Check file permissions
- Verify paths match between containers
- Check Unpackerr logs: `docker logs unpackerr`

### Tdarr

**Transcoding not working:**
```bash
# Check GPU access
docker exec tdarr ls -la /dev/dri

# Check logs
docker logs tdarr

# Verify temp folder has space
df -h | grep transcode
```

**Slow transcoding:**
- Enable hardware acceleration in Tdarr plugins
- Check CPU/GPU usage: `docker stats tdarr`
- Limit concurrent transcodes in Tdarr settings

### Bazarr

**Subtitles not downloading:**
- Add providers: Settings → Providers
- Check API limits (many subtitle sites have limits)
- Verify Sonarr/Radarr connections

---

## Performance Issues

### High CPU Usage

```bash
# Check which container
docker stats

# Common causes:
# - Tdarr transcoding (expected)
# - Jellyfin transcoding (use hardware acceleration)
# - Multiple downloads (qBittorrent)

# Limit CPU per container in docker-compose.yml:
services:
  tdarr:
    cpus: 4
```

### High Memory Usage

```bash
free -h
docker stats

# Limit memory in docker-compose.yml:
services:
  jellyfin:
    mem_limit: 4g
    mem_reservation: 2g
```

### Slow Network / Buffering

**For Jellyfin playback:**
1. Enable hardware transcoding
2. Adjust quality settings in Jellyfin client
3. Check network speed between client and server

**For downloads:**
1. Check VPN speed: https://www.speedtest.net/
2. Try different VPN server
3. Check qBittorrent connection limits

---

## Update Issues

### Container Won't Update

```bash
# Pull latest images
cd /opt/docker/media-stack
docker compose pull

# If specific image fails
docker pull lscr.io/linuxserver/jellyfin:latest

# Force recreate
docker compose up -d --force-recreate jellyfin
```

### Update Broke Configuration

**Rollback to previous version:**
```bash
# Check previous image
docker images

# Edit docker-compose.yml to use specific tag
jellyfin:
  image: lscr.io/linuxserver/jellyfin:10.8.13

# Deploy
docker compose up -d jellyfin
```

### Lost Data After Update

**Restore from backup:**
```bash
# Restore config backup
cd /opt/docker/media-stack
sudo tar -xzf backup-YYYYMMDD.tar.gz

# Restart services
docker compose down
docker compose up -d
```

---

## Emergency Recovery

### Complete Reset (Nuclear Option)

**WARNING: This will delete all configurations!**

```bash
# Backup first!
sudo tar -czf emergency-backup-$(date +%Y%m%d).tar.gz /opt/docker/media-stack

# Stop everything
cd /opt/docker/media-stack
docker compose down -v

# Remove configs (keeps media)
sudo rm -rf /opt/docker/media-stack/*/config/*

# Recreate
docker compose up -d

# Reconfigure from scratch
```

### Restore from Proxmox Backup

1. In Proxmox: VM → Backup → Restore
2. Select backup date
3. Restore to new VM or overwrite
4. Boot and verify

---

## Getting Help

### Collect Diagnostic Information

```bash
# System info
uname -a
cat /etc/os-release

# Docker info
docker --version
docker compose version

# Container status
docker ps -a

# Logs bundle
mkdir ~/debug-logs
docker logs gluetun > ~/debug-logs/gluetun.log
docker logs jellyfin > ~/debug-logs/jellyfin.log
docker logs radarr > ~/debug-logs/radarr.log
docker compose logs > ~/debug-logs/all-services.log

# Tar it up
cd ~
tar -czf debug-logs-$(date +%Y%m%d).tar.gz debug-logs/
```

### Resources

- **Jellyfin**: https://forum.jellyfin.org/
- **Servarr**: https://wiki.servarr.com/
- **Gluetun**: https://github.com/qdm12/gluetun/issues
- **TRaSH Guides**: https://trash-guides.info/
- **Reddit**:
  - r/jellyfin
  - r/radarr
  - r/sonarr
  - r/selfhosted

### Common Log Locations

```bash
# Docker container logs
docker logs <container_name>

# Application logs inside containers
docker exec jellyfin cat /config/logs/log_*.log
docker exec radarr cat /config/logs/radarr.txt
docker exec sonarr cat /config/logs/sonarr.txt

# System logs
sudo journalctl -u docker
sudo journalctl -xe
```

---

## Prevention Best Practices

1. **Regular Backups**
   ```bash
   # Weekly backup script
   #!/bin/bash
   tar -czf /mnt/media/backups/docker-$(date +%Y%m%d).tar.gz /opt/docker/media-stack
   ```

2. **Monitor Disk Space**
   ```bash
   # Add to crontab
   0 0 * * * df -h | grep -E 'Filesystem|/mnt/media' | mail -s "Disk Usage" you@email.com
   ```

3. **Keep Updated**
   ```bash
   # Update monthly
   cd /opt/docker/media-stack
   docker compose pull
   docker compose up -d
   docker system prune -a
   ```

4. **Test VPN Regularly**
   ```bash
   # Add to crontab to check hourly
   0 * * * * docker exec qbittorrent curl https://am.i.mullvad.net/connected | grep -q "connected" || echo "VPN DOWN" | mail -s "VPN Alert" you@email.com
   ```

---

Generated with Claude Code - https://claude.com/claude-code
