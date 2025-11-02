# Security Hardening Guide - Jellyfin Media Stack

## 🔒 Critical Security Notice

**This stack handles personal media and requires proper security measures. DO NOT expose these services directly to the internet without proper hardening.**

---

## Table of Contents
- [Security Overview](#security-overview)
- [Critical Security Measures](#critical-security-measures)
- [VPN Security](#vpn-security)
- [Service-Specific Security](#service-specific-security)
- [Network Security](#network-security)
- [Container Security](#container-security)
- [Secret Management](#secret-management)
- [Access Control](#access-control)
- [Monitoring & Auditing](#monitoring--auditing)
- [Security Checklist](#security-checklist)

---

## Security Overview

### Threat Model

**What we're protecting against:**
- IP address exposure while torrenting
- Unauthorized access to media and management interfaces
- Data breaches exposing API keys and credentials
- Container escape vulnerabilities
- Man-in-the-middle attacks

**What this stack does NOT protect against:**
- Physical access to the server
- Compromised client devices
- Social engineering attacks
- Zero-day vulnerabilities (until patched)

### Security Layers

```
Layer 1: Network (VPN, Firewall)
    ↓
Layer 2: Access Control (Authentication, Authorization)
    ↓
Layer 3: Application (API Keys, Secrets)
    ↓
Layer 4: Container (Isolation, User permissions)
    ↓
Layer 5: Host OS (Updates, Hardening)
```

---

## Critical Security Measures

### 1. VPN Kill Switch (CRITICAL)

**Why:** If VPN disconnects, your real IP could be exposed while torrenting.

**Test your kill switch:**
```bash
# Method 1: Stop Gluetun
docker stop gluetun

# Check qBittorrent - ALL downloads should stop immediately
# If they continue, YOUR IP IS LEAKING!

# Restart VPN
docker start gluetun
```

**Method 2: Verify network binding:**
```bash
# In qBittorrent WebUI:
# Tools → Options → Advanced → Network Interface
# MUST be set to: tun0

# Optional IP to bind: Leave EMPTY
```

**Automated monitoring:**
```bash
# Add to cron (check every 5 minutes)
*/5 * * * * docker exec qbittorrent curl -s https://am.i.mullvad.net/connected | grep -q "You are connected" || echo "VPN DOWN!" | mail -s "ALERT: VPN Disconnected" you@email.com
```

### 2. API Key Protection (CRITICAL)

**⚠️ WARNING:** API keys grant full control over services. Treat them like passwords!

**Best Practices:**
```bash
# NEVER commit .env to version control
echo ".env" >> .gitignore

# Set restrictive permissions
chmod 600 /opt/docker/media-stack/.env

# Rotate API keys if compromised
# In each *arr app: Settings → General → Security → Reset API Key
```

**What attackers can do with your API keys:**
- Download anything (using your VPN)
- Delete your entire library
- Access download history
- Change settings
- Add malicious indexers

### 3. Change Default Credentials (CRITICAL)

**qBittorrent:**
```bash
# Get default password
docker logs qbittorrent 2>&1 | grep "temporary password"

# Login immediately and change:
# WebUI → Tools → Options → Web UI → Authentication
# Change both username AND password
```

**All *arr Apps:**
```
Settings → General → Security
☑ Authentication: Forms (Login Page)
Username: [NOT "admin" - use something unique]
Password: [Strong password]
```

**Jellyfin:**
```
⚠️ CRITICAL: Admin access = shell access!
- Never share admin credentials
- Create limited user accounts for others
- Use strong passwords (20+ characters)
- Enable 2FA if using external access
```

### 4. Never Expose Services Directly (CRITICAL)

**❌ DO NOT DO THIS:**
```bash
# Port forwarding ALL services to the internet
Router: 8096 → 192.168.1.100:8096  # NO!
Router: 7878 → 192.168.1.100:7878  # NO!
Router: 8989 → 192.168.1.100:8989  # NO!
```

**✅ CORRECT APPROACH:**

**Option A: VPN Only (Most Secure)**
```
Use Tailscale, Wireguard, or OpenVPN
- Access media through VPN tunnel
- No ports exposed to internet
- End-to-end encryption
```

**Option B: Reverse Proxy + Auth (Secure if done right)**
```
Use Nginx Proxy Manager/Traefik/Caddy
+ Authelia/Authentik for SSO
+ Let's Encrypt SSL
+ Cloudflare (hides real IP)
+ Only expose ports 80/443
```

**Option C: Cloudflare Tunnel (Secure + Easy)**
```
- No port forwarding needed
- Free tier available
- DDoS protection included
- Easy SSL setup
```

---

## VPN Security

### Verify VPN is Working

**Test 1: IP Check**
```bash
# Your real IP (run on host)
curl https://ipinfo.io/ip

# qBittorrent's IP (should be DIFFERENT)
docker exec qbittorrent curl https://ipinfo.io/ip

# Should show Mullvad IP
docker exec qbittorrent curl https://am.i.mullvad.net/connected
```

**Test 2: DNS Leak**
```bash
# Check DNS servers being used
docker exec qbittorrent nslookup google.com

# Should show Mullvad DNS (10.64.0.1)
# NOT your ISP's DNS
```

**Test 3: WebRTC Leak**
```
1. Download a test torrent in qBittorrent
2. Go to https://ipleak.net
3. Check "Torrent Address detection"
4. Should show Mullvad IP, NOT your real IP
```

### VPN Configuration Hardening

**Mullvad-specific settings:**
```yaml
environment:
  # Use owned servers only (more trustworthy)
  - OWNED_ONLY=yes

  # Use servers that support port forwarding (better seeding)
  - SERVER_CITIES=Amsterdam,Stockholm,Copenhagen

  # Enable IPv6 leak protection
  - FIREWALL_VPN_INPUT_PORTS=

  # Regular server updates
  - UPDATER_PERIOD=24h
```

### Dealing with VPN Disconnections

**1. Monitor with Health Checks:**
```yaml
# Add to gluetun service in docker-compose.yml
healthcheck:
  test: ["CMD", "curl", "-f", "https://am.i.mullvad.net/connected"]
  interval: 60s
  timeout: 10s
  retries: 3
  start_period: 30s
```

**2. Restart Policy:**
```yaml
gluetun:
  restart: unless-stopped
```

**3. Alert on Failure:**
```bash
#!/bin/bash
# /usr/local/bin/vpn-monitor.sh

if ! docker exec gluetun curl -s https://am.i.mullvad.net/connected | grep -q "connected"; then
    # VPN is down - send alert
    echo "VPN disconnected at $(date)" | mail -s "VPN ALERT" you@email.com

    # Optionally stop qBittorrent to prevent leaks
    docker stop qbittorrent
fi
```

---

## Service-Specific Security

### Jellyfin Security

**⚠️ ADMIN ACCESS = SHELL ACCESS**

Granting administrator access to Jellyfin is equivalent to granting shell access to your server. An admin can:
- Delete files from your media storage
- Execute library scans that could overwrite data
- Install malicious plugins
- Access all user data

**Hardening Steps:**

1. **Create Limited Users**
```
Dashboard → Users → Add User
- Do NOT give admin privileges
- Restrict libraries as needed
- Enable parental controls if applicable
```

2. **Disable Unnecessary Features**
```
Dashboard → Networking
☐ Enable automatic port mapping
☐ Enable remote access (if not needed)
☑ Enable HTTPS (if using reverse proxy)
```

3. **Plugin Security**
```
- Only install plugins from official repository
- Review plugin permissions
- Keep plugins updated
- Remove unused plugins
```

4. **Hardware Access**
```
# Jellyfin needs GPU access for transcoding
# This is a security risk - container can access GPU
# Minimize by:
- Running as non-root (already configured)
- Using read-only mounts where possible
- Monitoring GPU usage for anomalies
```

### Radarr/Sonarr/Lidarr/Readarr Security

**1. Enable Authentication**
```
Settings → General → Security
Authentication: Forms (Login Page)
Username: [unique, not "admin"]
Password: [strong password]
```

**2. API Key Protection**
```
Settings → General → Security → API Key
- Treat as secret
- Never share publicly
- Rotate if compromised
- Store in password manager
```

**3. Download Client Configuration**
```
Settings → Download Clients
☑ Remove completed downloads: No (preserve for seeding)
☑ Check for finished downloads interval: 1 minute

# Security consideration:
# These apps can send ANY torrent to qBittorrent
# Ensure indexers are trusted
```

**4. Metadata Security**
```
Settings → Metadata
- Disable writing to media files (preserves originals)
- Use separate metadata folders
- Regular backups of metadata databases
```

### Prowlarr Security

**Most Critical Service** - Prowlarr has API keys for ALL your *arr apps!

**Hardening:**
```
1. Enable authentication (Settings → General → Security)
2. Use Flaresolverr for Cloudflare (not direct connections)
3. Only add trusted indexers
4. Review indexer privacy policies
5. Monitor indexer usage in logs
```

**Indexer Vetting:**
```
Before adding an indexer:
- Is it a known/established tracker?
- Does it require registration? (more accountable)
- Read privacy policy
- Check for HTTPS support
- Review user feedback
```

### qBittorrent Security

**Beyond VPN binding:**

1. **WebUI Access**
```
Options → Web UI
☑ IP address: 127.0.0.1 (bind to localhost)
☐ Use UPnP/NAT-PMP (disable)
☑ Enable Host header validation
☑ Enable clickjacking protection
☑ Enable CSRF protection
```

2. **Connection Limits**
```
Options → Connection
Global maximum connections: 500
Maximum per torrent: 100
Global maximum upload slots: 100
Maximum per torrent: 4

# Prevents resource exhaustion attacks
```

3. **Anonymous Mode**
```
Options → BitTorrent
☑ Enable anonymous mode
- Disables DHT, PeX, LSD when enabled
- Trade-off: Reduces discoverability but increases privacy
```

4. **IP Filtering** (Optional)
```
Options → Connection → IP Filtering
- Load blocklist of known malicious IPs
- Example: https://www.iblocklist.com/
- Warning: Can block legitimate peers
```

---

## Network Security

### Firewall Configuration

**On Proxmox Host:**
```bash
# Allow VM management
ufw allow 22/tcp     # SSH
ufw allow 8006/tcp   # Proxmox WebUI

# Everything else blocked by default
ufw enable
```

**On Ubuntu VM:**
```bash
# Install firewall
sudo apt install ufw

# Default deny
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow only what's needed
sudo ufw allow from 192.168.1.0/24 to any port 8096  # Jellyfin (LAN only)
sudo ufw allow from 192.168.1.0/24 to any port 7575  # Homarr (LAN only)
sudo ufw allow 22/tcp  # SSH (consider restricting to specific IPs)

# Port forwarding for qBittorrent (optional, for better seeding)
sudo ufw allow 8694/tcp
sudo ufw allow 8694/udp

# Enable
sudo ufw enable
```

### Docker Network Isolation

**Review network configuration:**
```bash
# List networks
docker network ls

# Inspect network
docker network inspect starr

# Verify services are on correct networks
docker inspect radarr | grep -A 10 Networks
```

**Best practices:**
- VPN'd services: Use `network_mode: "service:gluetun"`
- Media services: Separate bridge networks
- Database: Isolated network (e.g., `jellystat`)
- Never use `host` network mode (bypasses isolation)

### Port Exposure Review

**Currently exposed ports:**
```bash
# List all exposed ports
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Audit each port:
# - Is it necessary?
# - Should it be LAN-only?
# - Can it be behind reverse proxy?
```

**Minimize exposure:**
```yaml
# Instead of:
ports:
  - "8096:8096"  # Exposed to all interfaces

# Use (LAN only):
ports:
  - "192.168.1.100:8096:8096"  # Only accessible from LAN
```

---

## Container Security

### Run as Non-Root (Already Configured)

All containers use PUID/PGID 1000 (non-root). This is correct!

**Verify:**
```bash
docker exec jellyfin whoami
# Should show: abc (or similar non-root user)

docker exec jellyfin id
# Should show: uid=1000 gid=1000 (not uid=0)
```

### Container Updates

**Critical for security patches!**

```bash
# Weekly update routine
cd /opt/docker/media-stack

# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d

# Clean old images
docker image prune -a
```

**Automated updates (use with caution):**
```bash
# Watchtower for auto-updates
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower \
  --cleanup \
  --interval 86400  # Daily
```

### Resource Limits

**Prevent DoS through resource exhaustion:**

```yaml
# Add to docker-compose.yml for critical services
services:
  jellyfin:
    mem_limit: 4g
    mem_reservation: 2g
    cpus: 4.0
    pids_limit: 200
```

### Read-Only Filesystems (Advanced)

**For services that don't need write access:**
```yaml
services:
  someservice:
    read_only: true
    tmpfs:
      - /tmp
      - /var/tmp
```

**Warning:** Most services in this stack need write access. Only use for specific, stateless services.

### Security Scanning

**Scan images for vulnerabilities:**
```bash
# Using Trivy
docker run -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image ghcr.io/linuxserver/jellyfin:latest

# Using Docker Scout (built-in)
docker scout cves ghcr.io/linuxserver/jellyfin:latest
```

---

## Secret Management

### .env File Security

**Critical:**
```bash
# Set restrictive permissions
chmod 600 /opt/docker/media-stack/.env
chown mediaserver:mediaserver /opt/docker/media-stack/.env

# Never commit to git
echo ".env" >> .gitignore

# Backup encrypted
tar -czf env-backup.tar.gz .env
gpg -c env-backup.tar.gz  # Prompts for password
rm env-backup.tar.gz
```

### Secrets in Compose (Advanced)

**For production environments:**
```yaml
secrets:
  mullvad_key:
    file: ./secrets/mullvad_key.txt
  qbit_password:
    file: ./secrets/qbit_password.txt

services:
  gluetun:
    secrets:
      - mullvad_key
    environment:
      - WIREGUARD_PRIVATE_KEY_FILE=/run/secrets/mullvad_key
```

### Password Management

**Use a password manager for:**
- All service passwords
- API keys
- VPN credentials
- SSH keys

**Recommended:**
- Bitwarden (self-hosted or cloud)
- KeePassXC (offline)
- 1Password (commercial)

---

## Access Control

### SSH Hardening

**On the Ubuntu VM:**
```bash
sudo nano /etc/ssh/sshd_config

# Make these changes:
PermitRootLogin no
PasswordAuthentication no  # Use keys only
PubkeyAuthentication yes
Port 2222  # Non-standard port
AllowUsers mediaserver  # Only allow specific user

sudo systemctl restart sshd
```

**Use SSH keys:**
```bash
# On your client machine
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy to server
ssh-copy-id -p 2222 mediaserver@192.168.1.100

# Test (should work without password)
ssh -p 2222 mediaserver@192.168.1.100
```

### Fail2Ban

**Protect against brute force:**
```bash
sudo apt install fail2ban

# Create jail for SSH
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Monitor
sudo fail2ban-client status sshd
```

### Reverse Proxy Authentication (If Using)

**With Authelia or Authentik:**
```
Benefits:
- Single Sign-On (SSO) across all services
- Two-Factor Authentication (2FA)
- Password policies
- Session management
- Access logs

Setup:
- Beyond scope of this guide
- See: https://www.authelia.com/
- Or: https://goauthentik.io/
```

---

## Monitoring & Auditing

### Log Management

**Centralized logging:**
```bash
# View all logs in Dozzle
http://VM_IP:9999

# Or via command line
docker compose logs -f --tail=100

# Grep for security events
docker compose logs | grep -i "failed\|error\|unauthorized\|denied"
```

**Log rotation:**
```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Security Monitoring

**Things to monitor:**
```bash
# Failed login attempts
docker logs jellyfin | grep "Authentication failed"
docker logs radarr | grep "Authentication"

# Unusual download patterns
docker logs qbittorrent | grep "started"

# VPN connection status
docker logs gluetun | grep -i "disconnected\|error"

# API key usage
docker logs radarr | grep "API"
```

### Audit Checklist (Monthly)

```bash
#!/bin/bash
# /usr/local/bin/security-audit.sh

echo "=== Security Audit $(date) ===" > /tmp/audit.log

# Check for updates
echo "--- Available Updates ---" >> /tmp/audit.log
apt list --upgradable >> /tmp/audit.log 2>&1

# Check running containers
echo "--- Container Status ---" >> /tmp/audit.log
docker ps -a >> /tmp/audit.log

# Check exposed ports
echo "--- Exposed Ports ---" >> /tmp/audit.log
sudo netstat -tulpn | grep LISTEN >> /tmp/audit.log

# Check VPN status
echo "--- VPN Status ---" >> /tmp/audit.log
docker exec qbittorrent curl -s https://am.i.mullvad.net/connected >> /tmp/audit.log

# Check failed logins
echo "--- Failed Logins (Last 7 days) ---" >> /tmp/audit.log
sudo journalctl _SYSTEMD_UNIT=ssh.service | grep "Failed password" | tail -20 >> /tmp/audit.log

# Check disk usage
echo "--- Disk Usage ---" >> /tmp/audit.log
df -h >> /tmp/audit.log

# Send report
cat /tmp/audit.log | mail -s "Security Audit Report" you@email.com
```

---

## Security Checklist

### Initial Setup
- [ ] Changed all default passwords (qBittorrent, *arr apps, Jellyfin)
- [ ] Set strong passwords (20+ characters, unique)
- [ ] Enabled authentication on all *arr apps
- [ ] Set restrictive .env file permissions (chmod 600)
- [ ] Added .env to .gitignore
- [ ] Verified VPN kill switch works
- [ ] Tested VPN IP is different from real IP
- [ ] Configured firewall (ufw) on VM
- [ ] Disabled root SSH login
- [ ] Set up SSH key authentication
- [ ] Verified all containers run as non-root

### Network Security
- [ ] Services NOT exposed directly to internet
- [ ] Firewall only allows necessary ports
- [ ] VPN'd services bound to tun0 interface
- [ ] Port forwarding limited to qBittorrent only (8694)
- [ ] Tested DNS leak (should show Mullvad DNS)
- [ ] Tested WebRTC leak (should show Mullvad IP)

### Access Control
- [ ] Jellyfin admin account secured (strong password)
- [ ] Created limited Jellyfin user accounts
- [ ] Radarr/Sonarr use non-default usernames
- [ ] API keys stored securely (password manager)
- [ ] SSH hardened (keys only, non-standard port)
- [ ] Fail2ban configured

### Monitoring
- [ ] VPN monitoring script configured
- [ ] Log rotation enabled
- [ ] Regular log reviews scheduled
- [ ] Security audit script created
- [ ] Alert email configured
- [ ] Dozzle bookmarked for quick log access

### Maintenance
- [ ] Weekly container updates scheduled
- [ ] Monthly security audits scheduled
- [ ] Quarterly backup restores tested
- [ ] API key rotation policy established
- [ ] Incident response plan documented

### Advanced (Optional)
- [ ] Reverse proxy with SSL configured
- [ ] SSO (Authelia/Authentik) implemented
- [ ] Cloudflare Tunnel or Tailscale set up
- [ ] Container vulnerability scanning automated
- [ ] Intrusion detection system (IDS) deployed
- [ ] Security headers configured (CSP, HSTS, etc.)

---

## Incident Response

### If You Suspect a Breach

**1. Immediate Actions:**
```bash
# Stop all services
docker compose down

# Review logs for unusual activity
docker compose logs > /tmp/breach-logs.txt

# Check running processes
ps aux > /tmp/processes.txt

# Check network connections
sudo netstat -tunap > /tmp/connections.txt
```

**2. Investigation:**
- Review authentication logs
- Check for unauthorized changes
- Verify VPN was connected during incident
- Check torrent history for unexpected downloads
- Review API access logs

**3. Remediation:**
- Change all passwords
- Rotate all API keys
- Update all containers
- Scan for malware
- Review firewall rules
- Patch any vulnerabilities found

**4. Prevention:**
- Document what happened
- Implement additional controls
- Update this security guide
- Share lessons learned (anonymized)

---

## Additional Resources

**Security Guides:**
- OWASP Docker Security: https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html
- CIS Docker Benchmark: https://www.cisecurity.org/benchmark/docker
- Linux Server Security: https://madaidans-insecurities.github.io/guides/linux-hardening.html

**VPN Testing:**
- Mullvad Check: https://am.i.mullvad.net/
- IP Leak Test: https://ipleak.net/
- DNS Leak Test: https://www.dnsleaktest.com/

**Vulnerability Databases:**
- CVE Database: https://cve.mitre.org/
- Docker Hub Security: https://hub.docker.com/
- GitHub Security Advisories: https://github.com/advisories

---

**Last Updated:** 2025-10-24

**Generated with Claude Code** - https://claude.com/claude-code

**Remember:** Security is an ongoing process, not a one-time setup. Stay vigilant!
