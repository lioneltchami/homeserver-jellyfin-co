# Jellyfin Media Stack - Documentation Index

Welcome! This is your complete guide to setting up a professional media server on Proxmox.

## 🚨 IMPORTANT UPDATE (November 2025)

**Readarr has been retired** (July 2025) and is no longer maintained. See **[ERRATA.md](ERRATA.md)** for details and alternatives.

All other services are current and production-ready for 2025.

---

## Quick Navigation

**New to this?** Start with [QUICKSTART.md](QUICKSTART.md) for a condensed setup guide.

**Want every detail?** Read [README.md](README.md) for comprehensive step-by-step instructions.

**Hands-on person?** Use [SETUP_CHECKLIST.md](SETUP_CHECKLIST.md) to track your progress.

**Having problems?** Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for solutions.

---

## What is This Stack?

A complete, self-hosted media server solution that:
- Automatically downloads movies, TV shows, music, books, and comics
- Streams media to any device via Jellyfin
- Routes downloads through a VPN for privacy
- Manages user requests
- Optimizes storage with transcoding
- Provides a beautiful dashboard

**Total services:** 27 containers working together

**Estimated setup time:** 90 minutes (experienced) to 3 hours (beginners)

---

## Documentation Files

### Core Documentation

| File | Purpose | When to Use |
|------|---------|-------------|
| **[README.md](README.md)** | Complete setup guide with detailed explanations | Main reference, read through completely before starting |
| **[ERRATA.md](ERRATA.md)** | Important updates & corrections | 🚨 READ FIRST - Readarr retired July 2025 |
| **[QUICKSTART.md](QUICKSTART.md)** | Condensed setup steps for experienced users | Quick reference, assumes Docker/Linux knowledge |
| **[SETUP_CHECKLIST.md](SETUP_CHECKLIST.md)** | Interactive checklist to track progress | During setup to ensure nothing is missed |
| **[SECURITY.md](SECURITY.md)** | Security hardening and best practices | CRITICAL - Read before exposing to internet |
| **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** | Common problems and solutions | When things go wrong |
| **[DIRECTORY_STRUCTURE.md](DIRECTORY_STRUCTURE.md)** | Folder organization and paths | Understanding where everything goes |
| **[INDEX.md](INDEX.md)** | This file - navigation hub | Finding the right documentation |

### Configuration Files

| File | Purpose | Action Required |
|------|---------|-----------------|
| **docker-compose.yml** | Service definitions | Review, modify if needed |
| **.env.example** | Environment variable template | Copy to `.env` and fill in |
| **.env** | Your actual configuration | Create and keep private (DO NOT commit) |

---

## Setup Flow

```
1. Read INDEX.md (this file) ← You are here
         ↓
2. Review README.md (Overview & Architecture)
         ↓
3. Gather prerequisites (Hardware, Accounts, Downloads)
         ↓
4. Follow QUICKSTART.md OR README.md
         ↓
5. Use SETUP_CHECKLIST.md to track progress
         ↓
6. Reference DIRECTORY_STRUCTURE.md for paths
         ↓
7. Use TROUBLESHOOTING.md if issues arise
         ↓
8. Success! Enjoy your media server
```

---

## File Size & Reading Time

| File | Pages | Reading Time | Purpose |
|------|-------|--------------|---------|
| INDEX.md | 1 | 5 min | Navigation |
| QUICKSTART.md | 8 | 15 min | Fast setup |
| README.md | 35 | 45 min | Full guide |
| SETUP_CHECKLIST.md | 12 | 30 min | Progress tracking |
| SECURITY.md | 25 | 40 min | Security hardening |
| TROUBLESHOOTING.md | 20 | As needed | Problem solving |
| DIRECTORY_STRUCTURE.md | 10 | 15 min | Understanding structure |
| .env.example | 3 | 10 min | Configuration |

**Total reading time:** ~2.5 hours (but you can skim most)

---

## Who Should Use What?

### I'm brand new to Docker and Proxmox
1. Start with **README.md** - read EVERYTHING
2. Use **SETUP_CHECKLIST.md** - check off every step
3. Reference **DIRECTORY_STRUCTURE.md** - understand the layout
4. **Read SECURITY.md** - understand the risks
5. Keep **TROUBLESHOOTING.md** open in another tab

**Estimated time:** 3-4 hours

### I know Docker but new to media servers
1. Skim **README.md** sections 1-3 (Overview, Services, Architecture)
2. **Read SECURITY.md** - critical security measures
3. Follow **QUICKSTART.md** for fast setup
4. Reference **TROUBLESHOOTING.md** if needed
5. Use **DIRECTORY_STRUCTURE.md** for path questions

**Estimated time:** 1.5-2 hours

### I've set up *arr stacks before
1. Jump to **QUICKSTART.md**
2. Review **.env.example** and fill in **.env**
3. Run `docker compose up -d`
4. Configure services (you know the drill)

**Estimated time:** 45-90 minutes

### I just want to see what this is
1. Read **README.md** - Overview & Services sections
2. Check **DIRECTORY_STRUCTURE.md** - Port Reference table
3. Review **docker-compose.yml** - see what's included

**Estimated time:** 15 minutes

---

## Key Concepts

### Network Architecture
```
Internet → Mullvad VPN (Gluetun) → Download Services
                                    ↓
                              qBittorrent, Prowlarr, etc.

Local Network → Direct → Media Services
                         ↓
                    Jellyfin, *arr apps, etc.
```

**Why?** Downloads are private (VPN), but local streaming is fast (direct).

### Directory Mapping
```
Host System          Docker Container
-----------          ----------------
/opt/docker/...  →   /config          (app configs)
/mnt/media/      →   /data or /media  (media files)
```

**Why?** Consistent paths across all containers prevents import issues.

### Service Dependencies
```
Gluetun (VPN)
    ↓
qBittorrent, Prowlarr, Autobrr, FlareSolverr
    ↓
Radarr, Sonarr, Lidarr, Readarr, Kapowarr
    ↓
Jellyfin
```

**Why?** Each layer depends on the previous one working correctly.

---

## Common Questions

### Q: Do I need all these services?
**A:** No! Core services are: Jellyfin, Radarr, Sonarr, Prowlarr, qBittorrent, Gluetun. Others are optional enhancements.

### Q: Can I use a different VPN?
**A:** Yes! Gluetun supports 50+ providers. See: https://github.com/qdm12/gluetun-wiki

### Q: What if I don't have a NAS?
**A:** Use local disks. See README.md Part 1, Step 1.5 - Option B.

### Q: Can I run this on a different OS?
**A:** Yes, but this guide is Proxmox + Ubuntu specific. Adapt as needed.

### Q: How much does this cost?
**A:** Mullvad VPN: €5/month. Everything else is free and open-source.

### Q: Is this legal?
**A:** The software is legal. How you use it is your responsibility. Follow your local laws.

### Q: How much technical knowledge do I need?
**A:** Basic Linux command line, basic networking, ability to follow instructions carefully.

### Q: What if I get stuck?
**A:** Check TROUBLESHOOTING.md first. Then search GitHub issues for each service. Community forums are also helpful.

---

## Service URLs Quick Reference

After setup, access your services here (replace `VM_IP` with your VM's IP):

### Essential Services
- **Jellyfin:** http://VM_IP:8096 (Media server)
- **Homarr:** http://VM_IP:7575 (Dashboard - start here!)
- **Jellyseerr:** http://VM_IP:5055 (Request media)

### Management (*arr stack)
- **Radarr:** http://VM_IP:7878 (Movies)
- **Sonarr:** http://VM_IP:8989 (TV Shows)
- **Prowlarr:** http://VM_IP:9696 (Indexers)

### Downloads
- **qBittorrent:** http://VM_IP:8080 (Torrent client)

### Utilities
- **Dozzle:** http://VM_IP:9999 (Log viewer)
- **Tdarr:** http://VM_IP:8265 (Transcoding)
- **Bazarr:** http://VM_IP:6767 (Subtitles)

### Optional Services
- **Lidarr:** http://VM_IP:8686 (Music)
- **Readarr:** http://VM_IP:8787 (Books)
- **Kapowarr:** http://VM_IP:5757 (Comics)
- **Autobrr:** http://VM_IP:7474 (IRC grabber)
- **Wizarr:** http://VM_IP:5690 (Invites)
- **Jellystat:** http://VM_IP:3000 (Statistics)
- **Profilarr:** http://VM_IP:6868 (Profiles)

---

## Prerequisites Summary

Before starting, ensure you have:

### Accounts & Downloads
- [ ] Mullvad VPN account (https://mullvad.net)
- [ ] Ubuntu Server 24.04 LTS ISO
- [ ] Proxmox VE 8.0+ access

### Hardware
- [ ] 4+ CPU cores available
- [ ] 16GB+ RAM available
- [ ] 100GB+ storage for configs
- [ ] Large storage for media (500GB+ recommended)
- [ ] Static IP address for VM

### Knowledge
- [ ] Basic Linux command line
- [ ] Basic networking (IP addresses, ports)
- [ ] Ability to SSH into servers
- [ ] Basic Docker concepts (helpful but not required)

### Access
- [ ] Proxmox web interface access
- [ ] Router admin access (for port forwarding)
- [ ] Storage system access (if using NAS)

---

## Support & Resources

### Official Documentation
- **Jellyfin:** https://jellyfin.org/docs/
- **Servarr Wiki:** https://wiki.servarr.com/
- **Gluetun:** https://github.com/qdm12/gluetun-wiki
- **TRaSH Guides:** https://trash-guides.info/ ⭐ Highly recommended!

### Community
- **r/jellyfin:** Reddit community for Jellyfin
- **r/radarr & r/sonarr:** Media management help
- **r/selfhosted:** General self-hosting community
- **Discord servers:** Most projects have active Discord communities

### This Project
- **Issues:** [Report issues here]
- **Discussions:** [Ask questions here]
- **Updates:** [Check for updates here]

---

## Version Information

**Stack Version:** 1.0

**Last Updated:** 2025-10-20

**Tested On:**
- Proxmox VE 8.2
- Ubuntu Server 24.04 LTS
- Docker CE 27.x
- Docker Compose 2.x

**Service Versions:** All use `:latest` tags - auto-update

---

## Changelog

### 1.0 (2025-10-20)
- Initial comprehensive documentation
- Full Proxmox setup guide
- 27 services configured
- Complete troubleshooting guide
- Interactive setup checklist

---

## Contributing

Found an error? Have a suggestion? Want to add a service?
- Open an issue
- Submit a pull request
- Share your improvements

---

## License

This documentation and configuration is provided as-is for educational purposes.

All included services are open-source with their respective licenses:
- Jellyfin: GPLv2
- *arr stack: GPLv3
- Gluetun: MIT
- Others: See individual project licenses

---

## Credits

**Created by:** Claude Code (AI Assistant)

**Based on community knowledge from:**
- r/selfhosted
- TRaSH Guides
- Servarr Wiki
- Individual project documentation

**Special thanks to:**
- All open-source contributors
- The selfhosted community
- You, for choosing to self-host!

---

## Next Steps

1. **Choose your path:**
   - Beginner → [README.md](README.md)
   - Experienced → [QUICKSTART.md](QUICKSTART.md)

2. **Open SETUP_CHECKLIST.md** in a separate window

3. **Start your journey!**

---

**Remember:**
- Take your time
- Read error messages carefully
- Check TROUBLESHOOTING.md when stuck
- Join the community if you need help
- Have fun building your media empire!

---

Generated with Claude Code - https://claude.com/claude-code

**Documentation Generation Date:** October 20, 2025
