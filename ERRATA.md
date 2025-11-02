# Errata & Important Updates

## 🚨 Critical Update - November 2025

### Readarr Retirement (July 2025)

**IMPORTANT:** Readarr has been officially retired and is no longer maintained.

**Official Statement:**
> "The Readarr project has been retired due to the project's metadata becoming unusable, lack of time to remake or repair it, and the community effort to transition to using Open Library as the source stalling."

**Status:**
- ❌ LinuxServer repository archived (July 6, 2025)
- ❌ No official updates or support
- ❌ Metadata sources broken
- ⚠️ Third-party metadata mirrors exist (use at own risk)

**What You Should Do:**

### Option 1: Remove Readarr (Recommended)

If books are not critical to your setup:

```yaml
# In docker-compose.yml, comment out or delete:
# readarr:
#   image: lscr.io/linuxserver/readarr:0.4.19-nightly
#   ... (rest of config)
```

Also remove from `.env`:
```bash
# READARR_KEY=  # Not needed
```

### Option 2: Replace with LazyLibrarian

LazyLibrarian is an active alternative for book management:

```yaml
# Replace readarr service in docker-compose.yml with:
lazylibrarian:
  image: lscr.io/linuxserver/lazylibrarian:latest
  container_name: lazylibrarian
  environment:
    - PUID=${PUID}
    - PGID=${PGID}
    - TZ=${TZ}
  volumes:
    - ${BASE_PATH}/lazylibrarian/config:/config
    - ${MEDIA_SHARE}:/data
  ports:
    - 5299:5299
  networks:
    - proxy
    - starr
  restart: unless-stopped
```

**LazyLibrarian Access:** http://VM_IP:5299

### Option 3: Use Readarr with Third-Party Metadata (Advanced)

Some community members maintain metadata mirrors. **This is unsupported and at your own risk.**

Popular mirror: "rreading-glasses" (search community forums for setup)

---

## ⚠️ Service Clarifications

### FlareSolverr - Experimental vs Official

**Current Configuration (Experimental):**
```yaml
image: alexfozor/flaresolverr:pr-1300-experimental
```

**Issue:** This is a community fork with experimental features (nodriver).

**Recommended (Official):**
```yaml
flaresolverr:
  image: ghcr.io/flaresolverr/flaresolverr:latest
  container_name: flaresolverr
  network_mode: "service:gluetun"
  environment:
    - LOG_LEVEL=info
    - LOG_HTML=false
    - CAPTCHA_SOLVER=none
    - TZ=${TZ}
    - LANG=en_GB
    - DRIVER=nodriver  # Set to 'nodriver' or 'undetected-chromedriver'
  restart: unless-stopped
```

**Trade-off:**
- **Experimental:** May have better Cloudflare bypass, less stable
- **Official:** More stable, still effective

---

## ✅ Validated Services (November 2025)

These services are confirmed active and maintained:

| Service | Last Verified | Status |
|---------|---------------|--------|
| Jellyfin | Nov 2025 | ✅ Active |
| Radarr | Nov 2025 | ✅ Active |
| Sonarr | Nov 2025 | ✅ Active |
| Lidarr | Nov 2025 | ✅ Active |
| Kapowarr | Nov 1, 2025 | ✅ Active (updated yesterday!) |
| Prowlarr | Nov 2025 | ✅ Active |
| qBittorrent | Nov 2025 | ✅ Active |
| Gluetun | Nov 2025 | ✅ Active |
| Bazarr | Nov 2025 | ✅ Active |
| Tdarr | Nov 2025 | ✅ Active |
| Jellyseerr | Nov 2025 | ✅ Active |
| All others | Nov 2025 | ✅ Validated |

---

## 📋 Optional Services to Consider Skipping

These services are less established or have unclear purpose:

### Can Skip:
- **Dispatcharr** - Purpose unclear, optional
- **Profilarr** - Less established, optional
- **Janitorr** - Beta quality, can use Decluttarr instead

### Should Keep:
- Everything else is well-established and recommended

---

## 🔄 Migration Path from Original Stack

If you already deployed with Readarr:

1. **Backup Readarr database:**
```bash
docker exec readarr tar -czf /config/backup.tar.gz /config
docker cp readarr:/config/backup.tar.gz ~/readarr-backup.tar.gz
```

2. **Export your book list** (if possible via Readarr UI)

3. **Stop and remove Readarr:**
```bash
docker compose stop readarr
docker compose rm readarr
```

4. **Deploy LazyLibrarian** (if using Option 2)

5. **Import books manually into LazyLibrarian**

---

## 📅 Last Updated

**Date:** November 2, 2025
**Reviewer:** Claude Code
**Verification:** Web search + community forums

---

## 🔗 Resources

- **Readarr Retirement Notice:** https://wiki.servarr.com/readarr/faq
- **LazyLibrarian:** https://lazylibrarian.gitlab.io/
- **LazyLibrarian vs Readarr:** https://shareconnector.net/lazylibrarian-vs-readarr/
- **FlareSolverr Official:** https://github.com/FlareSolverr/FlareSolverr
- **Kapowarr:** https://github.com/Casvt/Kapowarr

---

Generated with Claude Code - https://claude.com/claude-code
