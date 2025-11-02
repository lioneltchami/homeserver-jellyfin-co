# Cloudflare Tunnel Setup Guide

## Overview

This guide shows you how to add Cloudflare Tunnel to your media stack for secure remote access **without port forwarding**.

**Last Updated:** November 2, 2025

---

## ✅ What Works with Cloudflare Tunnel

### **Services You CAN Expose (Not Behind VPN):**

| Service | URL Example | Recommended? |
|---------|-------------|--------------|
| **Jellyfin** | `jellyfin.yourdomain.com` | ✅ **YES** - Primary use case |
| **Jellyseerr** | `requests.yourdomain.com` | ✅ **YES** - User requests |
| **Homarr** | `dashboard.yourdomain.com` | ✅ **YES** - Central dashboard |
| **Wizarr** | `invite.yourdomain.com` | ✅ **YES** - Invite system |
| **Radarr** | `radarr.yourdomain.com` | ⚠️ **MAYBE** - If you manage remotely |
| **Sonarr** | `sonarr.yourdomain.com` | ⚠️ **MAYBE** - If you manage remotely |
| **Lidarr** | `lidarr.yourdomain.com` | ⚠️ **MAYBE** - If you manage remotely |
| **Bazarr** | `bazarr.yourdomain.com` | ⚠️ **MAYBE** - Subtitle management |
| **Jellystat** | `stats.yourdomain.com` | ⚠️ **MAYBE** - Statistics |

### **Services You CANNOT Expose (Behind VPN):**

❌ **qBittorrent** - Behind Gluetun, not accessible to cloudflared
❌ **Prowlarr** - Behind Gluetun, not accessible to cloudflared
❌ **Autobrr** - Behind Gluetun, not accessible to cloudflared
❌ **FlareSolverr** - Behind Gluetun, not accessible to cloudflared
❌ **Kapowarr** - Behind Gluetun, not accessible to cloudflared

**Why?** These services use `network_mode: "service:gluetun"` which isolates them from other Docker networks. Cloudflared cannot reach them.

**Is this a problem?** **NO!** You should **NEVER** expose download clients or indexers to the internet anyway. This is actually the correct security posture.

---

## 🔐 Security Consideration

**NEVER expose these services to the internet:**
- ❌ qBittorrent (download client)
- ❌ Prowlarr (indexer manager)
- ❌ Dozzle (log viewer - contains sensitive info)
- ❌ Any service that manages downloads or API keys

**Safe to expose (with authentication):**
- ✅ Jellyfin (media streaming)
- ✅ Jellyseerr (user requests with built-in auth)
- ✅ Homarr (dashboard with auth)

---

## 📋 Prerequisites

1. **Cloudflare account** (free tier works)
2. **Domain name** added to Cloudflare
3. **Existing media stack** deployed and working
4. **Terminal access** to your Proxmox VM

---

## 🚀 Setup Instructions

### **Step 1: Install Cloudflared on Your VM**

SSH into your Proxmox VM:

```bash
# Download cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Install it
sudo dpkg -i cloudflared-linux-amd64.deb

# Verify installation
cloudflared --version
```

### **Step 2: Authenticate with Cloudflare**

```bash
# This will open a browser for authentication
cloudflared tunnel login
```

**Follow the browser prompts:**
1. Login to your Cloudflare account
2. Select your domain
3. Authorize cloudflared

**You'll see:** "You have successfully logged in"

### **Step 3: Create a Tunnel**

```bash
# Create tunnel named "media-server"
cloudflared tunnel create media-server

# Note the Tunnel ID shown (you'll need it)
# Example: Created tunnel media-server with id: abc123-def456-ghi789
```

### **Step 4: Create Tunnel Configuration File**

```bash
# Create config directory
sudo mkdir -p /etc/cloudflared

# Create config file
sudo nano /etc/cloudflared/config.yml
```

**Add this configuration:**

```yaml
tunnel: media-server  # Replace with your tunnel name
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json  # Replace <TUNNEL_ID>

ingress:
  # Jellyfin (Primary media server)
  - hostname: jellyfin.yourdomain.com
    service: http://localhost:8096
    originRequest:
      noTLSVerify: true

  # Jellyseerr (User requests)
  - hostname: requests.yourdomain.com
    service: http://localhost:5055
    originRequest:
      noTLSVerify: true

  # Homarr (Dashboard)
  - hostname: dashboard.yourdomain.com
    service: http://localhost:7575
    originRequest:
      noTLSVerify: true

  # Wizarr (Invites)
  - hostname: invite.yourdomain.com
    service: http://localhost:5690
    originRequest:
      noTLSVerify: true

  # Radarr (Optional - only if you need remote management)
  # - hostname: radarr.yourdomain.com
  #   service: http://localhost:7878
  #   originRequest:
  #     noTLSVerify: true

  # Sonarr (Optional - only if you need remote management)
  # - hostname: sonarr.yourdomain.com
  #   service: http://localhost:8989
  #   originRequest:
  #     noTLSVerify: true

  # Catch-all rule (required)
  - service: http_status:404
```

**Important:** Replace:
- `<TUNNEL_ID>` with your actual tunnel ID from Step 3
- `yourdomain.com` with your actual domain
- Uncomment optional services if needed

### **Step 5: Create DNS Records**

For each service you want to expose:

```bash
# Jellyfin
cloudflared tunnel route dns media-server jellyfin.yourdomain.com

# Jellyseerr
cloudflared tunnel route dns media-server requests.yourdomain.com

# Homarr
cloudflared tunnel route dns media-server dashboard.yourdomain.com

# Wizarr
cloudflared tunnel route dns media-server invite.yourdomain.com

# Add more as needed...
```

### **Step 6: Install Cloudflared as a Service**

```bash
# Install the service
sudo cloudflared service install

# Start the service
sudo systemctl start cloudflared

# Enable on boot
sudo systemctl enable cloudflared

# Check status
sudo systemctl status cloudflared
```

**You should see:** "Active: active (running)"

### **Step 7: Verify It's Working**

```bash
# Check tunnel status
cloudflared tunnel info media-server

# Check service logs
sudo journalctl -u cloudflared -f
```

**Test access:**
1. Open your browser
2. Go to `https://jellyfin.yourdomain.com`
3. You should see Jellyfin login page (via Cloudflare!)

---

## 🐋 Alternative: Docker Compose Method

If you prefer running cloudflared as a Docker container:

### **Add to your docker-compose.yml:**

```yaml
services:
  # ... existing services ...

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - proxy
      - starr
```

### **Get your tunnel token:**

```bash
# Create tunnel via dashboard
# Go to: https://one.dash.cloudflare.com/
# Zero Trust → Networks → Tunnels → Create a tunnel
# Copy the tunnel token

# Add to .env file
echo "CLOUDFLARE_TUNNEL_TOKEN=your_token_here" >> .env
```

### **Configure tunnel in Cloudflare Dashboard:**

1. Go to: https://one.dash.cloudflare.com/
2. Zero Trust → Networks → Tunnels
3. Select your tunnel
4. Public Hostname → Add a public hostname

**For each service:**
- **Subdomain:** `jellyfin`
- **Domain:** `yourdomain.com`
- **Service:** `http://jellyfin:8096`

Repeat for other services (jellyseerr, homarr, etc.)

**Important:** Use container names (e.g., `jellyfin`, `jellyseerr`) NOT `localhost` when using Docker method.

---

## 🔒 Security Hardening

### **1. Enable Cloudflare Access (Zero Trust)**

Add authentication to your tunnels:

1. Go to: https://one.dash.cloudflare.com/
2. Zero Trust → Access → Applications → Add an application
3. **Self-hosted application**
4. **Application domain:** `jellyfin.yourdomain.com`
5. **Policy:** Allow only your email or GitHub account
6. Save

Now users must authenticate before accessing Jellyfin!

### **2. Restrict by Country**

In Cloudflare Access policies:
- Include: Countries you travel to
- Exclude: All others

### **3. Enable WAF Rules**

1. Cloudflare Dashboard → Security → WAF
2. Create rules to block common attacks
3. Enable Cloudflare Bot Fight Mode

### **4. Still Use Service Authentication**

Even with Cloudflare Access, enable authentication in:
- Jellyseerr (Settings → Users)
- Radarr (Settings → General → Authentication)
- Sonarr (Settings → General → Authentication)
- Homarr (Settings → Authentication)

**Defense in depth!**

---

## 🎯 Recommended Tunnel Configuration

### **For Personal Use:**

**Expose:**
- ✅ Jellyfin (streaming)
- ✅ Jellyseerr (requests)
- ⚠️ Homarr (dashboard) - only if you need remote management

**Don't expose:**
- ❌ Radarr/Sonarr (manage locally or via VPN)
- ❌ Dozzle (sensitive logs)
- ❌ qBittorrent, Prowlarr (can't expose + shouldn't expose)

### **For Sharing with Friends/Family:**

**Expose:**
- ✅ Jellyfin (they need this)
- ✅ Jellyseerr (they request content)
- ✅ Wizarr (they get invite links)

**Don't expose:**
- ❌ Everything else (they don't need management access)

---

## 🔧 Troubleshooting

### **Tunnel won't start:**

```bash
# Check config syntax
cloudflared tunnel validate /etc/cloudflared/config.yml

# Check credentials
ls -la /root/.cloudflared/

# Check logs
sudo journalctl -u cloudflared -n 50
```

### **Can't access service:**

```bash
# Test locally first
curl http://localhost:8096

# Check if service is running
docker ps | grep jellyfin

# Check tunnel status
cloudflared tunnel info media-server
```

### **DNS not resolving:**

```bash
# Check DNS records
nslookup jellyfin.yourdomain.com

# Should show Cloudflare IPs, not your home IP
```

### **Services behind VPN not accessible:**

**This is expected!** Services using `network_mode: "service:gluetun"` cannot be accessed by cloudflared.

**Solution:** Don't expose them (they shouldn't be exposed anyway).

---

## ⚡ Performance Considerations

### **Latency:**

Cloudflare Tunnel adds ~20-50ms latency compared to direct access.

**For streaming:** This is negligible, Jellyfin handles it well.

### **Bandwidth:**

Cloudflare's free tier has no bandwidth limits for Tunnel.

**But:** Aggressive streaming (4K, multiple users) may trigger rate limiting.

**Solution:** Use Cloudflare Stream or upgrade to paid plan if needed.

---

## 📊 Monitoring Your Tunnel

### **Check Tunnel Health:**

```bash
# Via CLI
cloudflared tunnel info media-server

# Via Dashboard
# https://one.dash.cloudflare.com/
# Networks → Tunnels → Your tunnel
```

### **View Analytics:**

1. Cloudflare Dashboard → Analytics
2. See requests, bandwidth, errors
3. Monitor for unusual activity

---

## 🔄 Updating Cloudflared

### **If installed as system service:**

```bash
# Download latest
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Install update
sudo dpkg -i cloudflared-linux-amd64.deb

# Restart service
sudo systemctl restart cloudflared
```

### **If running in Docker:**

```bash
cd /opt/docker/media-stack
docker compose pull cloudflared
docker compose up -d cloudflared
```

---

## 🎓 Advanced: Multiple Tunnels

You can create separate tunnels for different purposes:

```bash
# Media tunnel (Jellyfin, Jellyseerr)
cloudflared tunnel create media

# Management tunnel (Radarr, Sonarr)
cloudflared tunnel create management

# Each has its own config and DNS records
```

**Benefit:** Easier to manage, can disable management tunnel when not needed.

---

## 📚 Additional Resources

- **Official Docs:** https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **Tunnel Dashboard:** https://one.dash.cloudflare.com/
- **Community:** https://community.cloudflare.com/c/developers/cloudflare-tunnel/

---

## ✅ Final Checklist

After setup, verify:

- [ ] Tunnel is running: `systemctl status cloudflared`
- [ ] DNS records created for all services
- [ ] Can access Jellyfin via `https://jellyfin.yourdomain.com`
- [ ] Can access Jellyseerr via `https://requests.yourdomain.com`
- [ ] Cloudflare Access enabled (if using)
- [ ] Service-level authentication still enabled
- [ ] No attempts to expose VPN'd services (qBit, Prowlarr)
- [ ] Firewall still enabled (tunnel doesn't require open ports!)
- [ ] VPN still working (test qBittorrent downloads)

---

## 🎯 Summary

**What you can do with Cloudflare Tunnel:**
✅ Access Jellyfin from anywhere
✅ Let friends/family stream your content
✅ Request media via Jellyseerr remotely
✅ Manage your server via Homarr
✅ No port forwarding needed
✅ SSL included automatically
✅ DDoS protection from Cloudflare
✅ Optional authentication via Cloudflare Access

**What you cannot do:**
❌ Expose qBittorrent (behind VPN + shouldn't expose)
❌ Expose Prowlarr (behind VPN + shouldn't expose)
❌ Bypass VPN for download services (good thing!)

**The bottom line:** Cloudflare Tunnel works perfectly for your use case. The services users actually need (Jellyfin, Jellyseerr) are fully compatible. The services behind VPN (qBittorrent, Prowlarr) can't be exposed, but **you shouldn't expose them anyway!**

---

Generated with Claude Code - https://claude.com/claude-code

**Last Updated:** November 2, 2025
