# Jelly Media Stack

Dockerized Jellyfin + request/download automation (Jellyseerr, Sonarr, Radarr, Prowlarr, Transmission via VPN), reverse proxy (Nginx Proxy Manager), and monitoring (Uptime Kuma).

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

## Service endpoints (LAN defaults)
- Jellyfin: `http://<host>:8096`
- Jellyseerr: `http://<host>:5055`
- Sonarr: `http://<host>:8989`
- Radarr: `http://<host>:7878`
- Prowlarr: `http://<host>:9696`
- Transmission (VPN): `http://<host>:9091`
- Nginx Proxy Manager: `http://<host>:8181`
- Uptime Kuma: `http://<host>:3001`

## Notes
- Media/download paths come from `.env` (`MEDIA_PATH`, `DOWNLOADS_PATH`); defaults point to `/mnt/media-1tb`.
- Config/state persists under `./configs/...` and `./homarr/appdata/...` (directories are auto-created by Compose).
- Jellyfin uses `runtime: nvidia`; adjust/remove if GPU is unavailable.
- Keep API keys and passwords out of git—`.env` is ignored; rotate secrets if you previously committed them elsewhere.
