# Jelly Media Stack

Dockerized Jellyfin + request/download automation (Jellyseerr, Sonarr, Radarr, Prowlarr, Transmission via VPN), reverse proxy (Nginx Proxy Manager), and monitoring (Uptime Kuma).

**Repository:** `binoy-d/jelly-media-stack`  
**Current Branch:** `main`  
**Production Directory:** `/home/daniel/jelly-stack`

## 📋 Table of Contents
- [Quick Start](#quick-start-few-steps)
- [Architecture Overview](#architecture-overview)
- [Service Details](#service-details)
- [Environment Variables](#environment-variables)
- [Network Configuration](#network-configuration)
- [Cloudflare Tunnel Integration](#cloudflare-tunnel-integration)
- [Directory Structure](#directory-structure)
- [Docker Commands](#docker-commands)
- [Troubleshooting](#troubleshooting)
- [Security & Secrets](#security--secrets)
- [Maintenance](#maintenance)

## Quick start (few steps)
1) Install Docker & Docker Compose v2; ensure GPU runtime if you want Jellyfin hardware accel.
2) Clone this repo and enter it:
   ```bash
   git clone git@github.com:binoy-d/jelly-media-stack.git
   cd jelly-media-stack
   ```
3) Copy env template and fill secrets/paths:
   ```bash
   cp .env.example .env
   # edit .env with your VPN creds, media/download paths, timezone, etc.
   ```
4) Add VPN files for Transmission to `configs/vpn-transmission/`:
   - `<config>.ovpn` (e.g., `cyberghost.ovpn`)
   - `ca.crt`, `client.crt`, `client.key`
   - (Optional) `openvpn-credentials.txt` if your provider uses it; match the path inside the `.ovpn`.
5) Launch the stack:
   ```bash
   docker compose up -d
   docker compose ps
   ```

## Quick start (few steps)
1) Install Docker & Docker Compose v2; ensure GPU runtime if you want Jellyfin hardware accel.
2) Clone this repo and enter it:
   ```bash
   git clone git@github.com:binoy-d/jelly-media-stack.git
   cd jelly-media-stack
   ```
3) Copy env template and fill secrets/paths:
   ```bash
   cp .env.example .env
   # edit .env with your VPN creds, media/download paths, timezone, etc.
   ```
4) Add VPN files for Transmission to `configs/vpn-transmission/`:
   - `<config>.ovpn` (e.g., `cyberghost.ovpn`)
   - `ca.crt`, `client.crt`, `client.key`
   - (Optional) `openvpn-credentials.txt` if your provider uses it; match the path inside the `.ovpn`.
5) Launch the stack:
   ```bash
   docker-compose up -d
   docker-compose ps
   ```

## Architecture Overview

### System Architecture
```
Internet → Cloudflare Tunnel (cloudflared) → Local Services
                                            ├─ Jellyfin (8096)
                                            ├─ Jellyseerr (5055)
                                            ├─ Sonarr (8989)
                                            ├─ Radarr (7878)
                                            ├─ Prowlarr (9696)
                                            ├─ Transmission (9091)
                                            ├─ Uptime Kuma (3001)
                                            └─ Nginx Proxy Manager (8181)
```

### Data Flow
1. **Media Requests**: Users request media via Jellyseerr (see.binoy.co)
2. **Indexer Management**: Prowlarr manages torrent/usenet indexers
3. **Content Acquisition**: 
   - Sonarr (TV shows) searches indexers and sends to Transmission
   - Radarr (Movies) searches indexers and sends to Transmission
4. **Download**: Transmission downloads via VPN to `/mnt/media-1tb/downloads`
5. **Media Library**: Downloaded content moved to `/mnt/media-1tb/` for Jellyfin
6. **Streaming**: Jellyfin serves media to users (jelly.binoy.co)

## Service Details

### Jellyfin (Media Server)
- **Port**: 8096
- **Public URL**: https://jelly.binoy.co
- **Container**: `jellyfin`
- **GPU Acceleration**: Yes (NVIDIA runtime enabled)
- **Config**: `./configs/jellyfin/`
- **Media Path**: `/mnt/media-1tb/` (read-only mount)
- **Features**: Hardware transcoding, library management, user authentication

### Jellyseerr (Media Request Management)
- **Port**: 5055
- **Public URL**: https://see.binoy.co
- **Container**: `jellyseerr`
- **Config**: `./configs/jellyseerr/`
- **Purpose**: User-friendly interface for requesting TV shows and movies
- **Integration**: Connects to Sonarr, Radarr, and Jellyfin

### Sonarr (TV Show Management)
- **Port**: 8989
- **Public URL**: https://sonarr.binoy.co
- **Container**: `sonarr`
- **Config**: `./configs/sonarr/`
- **Media Path**: `/mnt/media-1tb/` (mounted as `/media`)
- **Download Path**: `/mnt/media-1tb/downloads` (mounted as `/data`)
- **Purpose**: Automates TV show downloads, renames files, manages library

### Radarr (Movie Management)
- **Port**: 7878
- **Public URL**: https://radarr.binoy.co
- **Container**: `radarr`
- **Config**: `./configs/radarr/`
- **Media Path**: `/mnt/media-1tb/` (mounted as `/media`)
- **Download Path**: `/mnt/media-1tb/downloads` (mounted as `/data`)
- **Purpose**: Automates movie downloads, renames files, manages library

### Prowlarr (Indexer Manager)
- **Port**: 9696
- **Public URL**: https://prowlarr.binoy.co
- **Container**: `prowlarr`
- **Config**: `./configs/prowlarr/`
- **Purpose**: Centralized indexer management for Sonarr and Radarr
- **Integration**: Syncs indexers to both Sonarr and Radarr automatically

### VPN Transmission (Torrent Client)
- **Port**: 9091
- **Public URL**: https://transmission.binoy.co
- **Container**: `vpn-transmission`
- **Image**: `haugene/transmission-openvpn`
- **Config**: `./configs/vpn-transmission/`
- **VPN Provider**: CyberGhost (CUSTOM configuration)
- **Download Path**: `/mnt/media-1tb/downloads`
- **Web UI**: Combustion
- **Security**: All traffic routed through VPN (kill switch enabled)
- **Network**: Requires `NET_ADMIN` capability and `/dev/net/tun` device

### Nginx Proxy Manager (Reverse Proxy)
- **Ports**: 
  - 8081 (HTTP)
  - 4443 (HTTPS)
  - 8181 (Admin UI)
- **Public URL**: https://nginx.binoy.co
- **Container**: `nginx-proxy-manager`
- **Config**: `./configs/npm/data/`
- **Certificates**: `./configs/npm/letsencrypt/`
- **Purpose**: SSL/TLS termination, reverse proxy for local services

### Uptime Kuma (Monitoring)
- **Port**: 3001
- **Public URL**: https://kuma.binoy.co
- **Container**: `uptime-kuma`
- **Config**: `./configs/uptime-kuma/`
- **Docker Socket**: Mounted for container monitoring
- **Purpose**: Monitor service availability and health

## Environment Variables

### Required Variables (`.env` file)
```bash
# User/Group IDs (use `id -u` and `id -g` to find yours)
PUID=1000                              # User ID for file permissions
PGID=1000                              # Group ID for file permissions
TZ=America/Chicago                     # Timezone for all services

# Storage Paths
MEDIA_PATH=/mnt/media-1tb              # Where media files are stored
DOWNLOADS_PATH=/mnt/media-1tb/downloads # Where downloads go

# VPN Configuration (for Transmission)
OPENVPN_PROVIDER=CUSTOM                # VPN provider type
OPENVPN_CONFIG=cyberghost              # OpenVPN config file name (without .ovpn)
OPENVPN_USERNAME=GGwEdJgJaB            # VPN username
OPENVPN_PASSWORD=j8M4CGznPv            # VPN password
LOCAL_NETWORK=192.168.86.0/24          # Your local network CIDR

# Transmission Configuration
TRANSMISSION_WEB_UI=combustion         # Web UI theme
TRANSMISSION_RPC_AUTHENTICATION_REQUIRED=true
TRANSMISSION_RPC_USERNAME=daniel       # Transmission login username
TRANSMISSION_RPC_PASSWORD=supersecurepassword # Transmission login password
```

### How to Set Environment Variables
1. Copy `.env.example` to `.env`
2. Edit values to match your setup
3. **Never commit `.env` to git** (already in `.gitignore`)
4. Restart containers after changing: `docker-compose restart`

## Network Configuration

### Local Network Access
All services are accessible on your local network at:
- Jellyfin: `http://192.168.86.x:8096`
- Jellyseerr: `http://192.168.86.x:5055`
- Sonarr: `http://192.168.86.x:8989`
- Radarr: `http://192.168.86.x:7878`
- Prowlarr: `http://192.168.86.x:9696`
- Transmission: `http://192.168.86.x:9091`
- Nginx Proxy Manager: `http://192.168.86.x:8181`
- Uptime Kuma: `http://192.168.86.x:3001`

### Docker Networking
- All containers run on the default bridge network
- Containers can communicate via container names (e.g., `http://jellyfin:8096`)
- VPN container has special network capabilities (`NET_ADMIN`)

## Cloudflare Tunnel Integration

### Tunnel Configuration
The stack is integrated with Cloudflare Tunnel (cloudflared) for secure remote access.

**Tunnel Config Location**: `/etc/cloudflared/config.yml`  
**Tunnel Name**: `jellyfin-tunnel`  
**Credentials**: `/etc/cloudflared/a507fd82-6791-4b47-b0ca-bedebb091152.json`

### Public URLs (via Cloudflare Tunnel)
```yaml
ingress:
  - hostname: jelly.binoy.co
    service: http://localhost:8096      # Jellyfin
  - hostname: see.binoy.co
    service: http://localhost:5055      # Jellyseerr
  - hostname: kuma.binoy.co
    service: http://localhost:3001      # Uptime Kuma
  - hostname: sonarr.binoy.co
    service: http://localhost:8989      # Sonarr
  - hostname: radarr.binoy.co
    service: http://localhost:7878      # Radarr
  - hostname: transmission.binoy.co
    service: http://localhost:9091      # Transmission
  - hostname: prowlarr.binoy.co
    service: http://localhost:9696      # Prowlarr
  - hostname: nginx.binoy.co
    service: http://localhost:8181      # Nginx Proxy Manager
  - service: http_status:404            # Catch-all
```

### Cloudflared Service Management
```bash
# Check status
systemctl status cloudflared

# Restart tunnel
sudo systemctl restart cloudflared

# View logs
sudo journalctl -u cloudflared -f

# Edit configuration
sudo nano /etc/cloudflared/config.yml
# Then restart: sudo systemctl restart cloudflared
```

### Adding New Routes
To expose a new service:
1. Add to `/etc/cloudflared/config.yml` ingress list
2. Restart cloudflared: `sudo systemctl restart cloudflared`
3. DNS is automatically configured by Cloudflare

## Directory Structure

```
jelly-stack/
├── .env                          # Environment variables (NEVER commit)
├── .env.example                  # Template for environment variables
├── .gitignore                    # Git ignore rules
├── README.md                     # This file
├── ai-context.md                 # AI agent context/notes
├── docker-compose.yml            # Docker services definition
└── configs/                      # Service configurations (ignored by git)
    ├── jellyfin/                 # Jellyfin config & metadata
    │   ├── config/               # Server configuration
    │   ├── data/                 # Database and cache
    │   ├── log/                  # Application logs
    │   ├── metadata/             # Media metadata/images
    │   ├── plugins/              # Jellyfin plugins
    │   └── root/                 # Root user data
    ├── jellyseerr/               # Jellyseerr database & settings
    │   ├── settings.json         # Application settings
    │   ├── cache/                # Cached data
    │   ├── db/                   # SQLite database
    │   └── logs/                 # Application logs
    ├── prowlarr/                 # Prowlarr config & database
    │   ├── config.xml            # Configuration
    │   ├── prowlarr.db           # SQLite database
    │   └── logs/                 # Application logs
    ├── sonarr/                   # Sonarr config & database
    │   ├── config.xml            # Configuration
    │   ├── sonarr.db             # SQLite database
    │   └── logs/                 # Application logs
    ├── radarr/                   # Radarr config & database
    │   ├── config.xml            # Configuration
    │   ├── radarr.db             # SQLite database
    │   └── logs/                 # Application logs
    ├── vpn-transmission/         # VPN & Transmission config
    │   ├── cyberghost.ovpn       # OpenVPN configuration
    │   ├── ca.crt                # Certificate Authority cert
    │   ├── client.crt            # Client certificate
    │   └── client.key            # Client private key
    ├── npm/                      # Nginx Proxy Manager
    │   ├── data/                 # Application data
    │   └── letsencrypt/          # SSL certificates
    └── uptime-kuma/              # Uptime Kuma monitoring
        └── kuma.db               # SQLite database
```

### Important Paths
- **Media Storage**: `/mnt/media-1tb/` (mounted drive)
- **Downloads**: `/mnt/media-1tb/downloads/`
- **Project Root**: `/home/daniel/jelly-stack/`
- **Old Setup**: `/home/daniel/jelly/` (deprecated, DO NOT USE)

## Docker Commands

### Starting the Stack
```bash
cd /home/daniel/jelly-stack
docker-compose up -d                    # Start all services in background
docker-compose ps                       # Check status
docker-compose logs -f                  # View all logs (follow mode)
```

### Managing Individual Services
```bash
docker-compose restart jellyfin         # Restart specific service
docker-compose stop sonarr              # Stop specific service
docker-compose start sonarr             # Start specific service
docker-compose logs -f radarr           # View logs for specific service
```

### Stopping the Stack
```bash
docker-compose down                     # Stop and remove containers
docker-compose down -v                  # Stop and remove volumes (DELETES DATA!)
docker-compose down --remove-orphans    # Stop and remove orphan containers
```

### Updating Services
```bash
docker-compose pull                     # Pull latest images
docker-compose up -d                    # Recreate containers with new images
docker image prune -a                   # Clean up old images
```

### Health Checks
```bash
docker-compose ps                       # Check container status
docker stats                            # View resource usage
docker-compose exec jellyfin bash       # Enter container shell
```

### Using docker vs docker-compose
- This system uses `docker-compose` (v1 syntax)
- Some systems use `docker compose` (v2 syntax)
- Both should work, but v1 is currently configured

## Troubleshooting

### Container Won't Start
```bash
# View logs
docker-compose logs <service-name>

# Check if port is already in use
sudo netstat -tulpn | grep <port>

# Force recreate container
docker-compose up -d --force-recreate <service-name>
```

### VPN Issues (Transmission)
```bash
# Check VPN connection
docker-compose logs vpn-transmission | grep -i "connection"

# Verify VPN files exist
ls -la configs/vpn-transmission/

# Check if traffic is going through VPN
docker-compose exec vpn-transmission curl ifconfig.me
```

### Permission Issues
```bash
# Fix ownership of configs
sudo chown -R 1000:1000 ./configs

# Fix ownership of media
sudo chown -R 1000:1000 /mnt/media-1tb

# Check current permissions
ls -la ./configs
```

### Network Issues
```bash
# Test service locally
curl http://localhost:8096

# Check if cloudflared is running
systemctl status cloudflared
ps aux | grep cloudflared

# Test public access
curl https://jelly.binoy.co
```

### Database Corruption
```bash
# Backup before attempting fixes
cp -r configs/sonarr configs/sonarr.backup

# Most *arr apps have built-in recovery
# Check logs for specific recovery steps
docker-compose logs sonarr | grep -i "database"
```

### Clean Restart
```bash
# Stop everything
docker-compose down

# Check for orphaned containers
docker ps -a

# Start fresh
docker-compose up -d

# Monitor startup
docker-compose logs -f
```

## Security & Secrets

### What's Ignored by Git
- `.env` - All environment variables and secrets
- `configs/` - Contains API keys, databases, and sensitive data
- Any service-specific credentials or tokens

### Secrets Management
1. **Never commit** `.env` file
2. **Never commit** `configs/` directory
3. VPN credentials are in `.env` only
4. Transmission password is in `.env`
5. Service API keys are stored in their respective config databases

### Rotating Secrets
If secrets are compromised:
1. Update `.env` with new credentials
2. Update VPN files if needed
3. Restart affected services: `docker-compose restart`
4. Update API keys in service UIs (Sonarr, Radarr, Prowlarr)

### Access Control
- Transmission has username/password authentication
- Jellyfin has user accounts with passwords
- Jellyseerr integrates with Jellyfin authentication
- Other *arr apps should be protected by Cloudflare Access or Nginx auth

## Maintenance

### Regular Tasks
```bash
# Update all containers (monthly)
cd /home/daniel/jelly-stack
docker-compose pull
docker-compose up -d
docker image prune -a

# Check disk space (weekly)
df -h /mnt/media-1tb

# Backup configs (weekly)
tar -czf jelly-stack-backup-$(date +%Y%m%d).tar.gz configs/

# Monitor logs for errors (as needed)
docker-compose logs --tail=100
```

### Backup Strategy
```bash
# Backup entire config directory
sudo tar -czf /home/daniel/backups/jelly-configs-$(date +%Y%m%d).tar.gz \
  /home/daniel/jelly-stack/configs/

# Backup specific service
tar -czf sonarr-backup-$(date +%Y%m%d).tar.gz configs/sonarr/

# Restore from backup
tar -xzf jelly-configs-20251127.tar.gz
```

### Monitoring Health
- Use Uptime Kuma to monitor all services
- Check https://kuma.binoy.co for status dashboard
- Set up notifications in Uptime Kuma for downtime alerts

### Disk Space Management
```bash
# Check download folder size
du -sh /mnt/media-1tb/downloads

# Find large files
find /mnt/media-1tb/downloads -type f -size +10G

# Clean old downloads (these should be auto-deleted by Sonarr/Radarr)
sudo rm -rf /mnt/media-1tb/downloads/tv-sonarr/*
sudo rm -rf /mnt/media-1tb/downloads/radarr/*
```

### ⚠️ IMPORTANT: Configure Sonarr/Radarr to Move Files (Not Copy)

By default, Sonarr and Radarr may COPY files instead of MOVE them, leaving duplicates in the downloads folder. This wastes massive amounts of space!

**Fix this in Sonarr:**
1. Go to https://sonarr.binoy.co
2. Settings → Media Management
3. **Enable**: "Use Hardlinks instead of Copy" ✓
4. **Enable**: "Delete empty folders" ✓
5. Under "Completed Download Handling":
   - **Enable**: "Remove Completed Downloads" ✓
   - This tells Sonarr to remove from download client after importing
6. Save changes
7. Restart Sonarr: `docker-compose restart sonarr`

**Fix this in Radarr:**
1. Go to https://radarr.binoy.co
2. Settings → Media Management
3. **Enable**: "Use Hardlinks instead of Copy" ✓
4. **Enable**: "Delete empty folders" ✓
5. Under "Completed Download Handling":
   - **Enable**: "Remove Completed Downloads" ✓
6. Save changes
7. Restart Radarr: `docker-compose restart radarr`

**Why Hardlinks?**
- Hardlinks create a reference to the same file data without copying
- File exists in both downloads/ and shows/movies/ but only uses space once
- When download is removed, the file stays in your library
- Saves tons of space!

**Verify it's working:**
```bash
# Downloads folder should stay small after this
watch -n 60 du -sh /mnt/media-1tb/downloads

# Check that files are hardlinked (same inode number)
ls -li /mnt/media-1tb/downloads/tv-sonarr/ShowName/
ls -li /mnt/media-1tb/shows/ShowName/
# If inode numbers match, it's a hardlink (good!)
```

## Advanced Configuration

### GPU Acceleration (Jellyfin)
The docker-compose.yml is configured for NVIDIA GPU:
```yaml
runtime: nvidia
deploy:
  resources:
    reservations:
      devices:
        - capabilities: [gpu]
```

If you don't have NVIDIA GPU, remove these lines from docker-compose.yml.

### Custom VPN Provider
To change VPN providers:
1. Update `.env` with new provider: `OPENVPN_PROVIDER=PROVIDERNAME`
2. See supported providers: https://haugene.github.io/docker-transmission-openvpn/supported-providers/
3. Or use `CUSTOM` and provide `.ovpn` files in `configs/vpn-transmission/`

### Adding New Services
1. Add service to `docker-compose.yml`
2. Add port mapping
3. Add volume mounts
4. Update this README
5. Add to Cloudflare tunnel config if needed public access
6. Restart cloudflared

## Git Workflow

### Current Setup
- **Repository**: https://github.com/binoy-d/jelly-media-stack
- **Branch**: `main`
- **Remote**: `origin` (git@github.com:binoy-d/jelly-media-stack.git)

### Making Changes
```bash
cd /home/daniel/jelly-stack

# Check status
git status

# Stage changes
git add docker-compose.yml README.md

# Commit
git commit -m "Description of changes"

# Push to GitHub
git push origin main
```

### NEVER Commit
- `.env` file
- `configs/` directory
- Any files with passwords, API keys, or personal data

## AI Agent Notes

### Key Information for Automation
- **Working Directory**: `/home/daniel/jelly-stack/`
- **Docker Command**: `docker-compose` (not `docker compose`)
- **All configs are persistent**: Data survives container restarts
- **VPN is critical**: Transmission will fail if VPN doesn't connect
- **Permissions matter**: UID/GID 1000:1000 for all file operations
- **Two locations exist**: 
  - ACTIVE: `/home/daniel/jelly-stack/` ✅
  - DEPRECATED: `/home/daniel/jelly/` ❌ DO NOT USE

### Common Operations
```bash
# Check if stack is running
cd /home/daniel/jelly-stack && docker-compose ps

# Restart entire stack
cd /home/daniel/jelly-stack && docker-compose restart

# View logs for debugging
cd /home/daniel/jelly-stack && docker-compose logs -f --tail=50

# Update and restart
cd /home/daniel/jelly-stack && docker-compose pull && docker-compose up -d
```

### Integration Points
- Prowlarr syncs to Sonarr/Radarr (must configure API keys in UI)
- Sonarr/Radarr send downloads to Transmission
- Transmission downloads to `/mnt/media-1tb/downloads`
- Sonarr/Radarr move completed files to `/mnt/media-1tb/`
- Jellyfin scans `/mnt/media-1tb/` for new media
- Jellyseerr sends requests to Sonarr/Radarr

### Critical Configuration Issues
**Downloads Folder Bloat (COMMON ISSUE):**
- Problem: Downloads folder can grow to 1TB+ if misconfigured
- Root Cause: Sonarr/Radarr set to COPY instead of HARDLINK/MOVE
- Solution: Enable "Use Hardlinks" and "Remove Completed Downloads" in Settings → Media Management
- Verification: Downloads folder should stay under 100GB normally
- Cleanup: `sudo rm -rf /mnt/media-1tb/downloads/{tv-sonarr,radarr}/*`
