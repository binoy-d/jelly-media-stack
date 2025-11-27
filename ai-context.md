# Jelly Stack AI Context

Quick orientation for agents working on this media stack repo (Dockerized) in this directory.

## Purpose
- Personal media server pipeline: Jellyseerr handles requests → Radarr/Sonarr fetch media → Transmission (via VPN) downloads → Jellyfin serves media. Prowlarr manages indexers, Nginx Proxy Manager handles reverse proxying, Uptime Kuma monitors uptime. Homarr data dirs included if you add that container.

## Core locations
- Compose file: `docker-compose.yml`.
- Persistent config dirs (gitignored): `configs/jellyfin`, `configs/jellyseerr`, `configs/prowlarr`, `configs/sonarr`, `configs/radarr`, `configs/vpn-transmission`, `configs/npm`, `configs/uptime-kuma`.
- Media + downloads: host paths come from `.env` (`MEDIA_PATH`, `DOWNLOADS_PATH`); defaults point to `/mnt/media-1tb` and `/mnt/media-1tb/downloads`.
- VPN/OpenVPN artifacts for Transmission: drop `*.ovpn`, `ca.crt`, `client.crt`, `client.key` into `configs/vpn-transmission/` (not tracked).
- Homarr data (if used): `homarr/appdata/{db,redis,trusted-certificates}` (gitignored).

## Services & ports (host)
- Jellyfin `8096` (expects NVIDIA GPU runtime).
- Jellyseerr `5055`.
- Prowlarr `9696`.
- Sonarr `8989`.
- Radarr `7878`.
- Transmission Web UI (via VPN) `9091`.
- Nginx Proxy Manager `8081:80`, `4443:443`, `8181:81` (admin UI on `8181`).
- Uptime Kuma `3001`.

## How to operate
- From repo root: `docker compose up -d` to start, `docker compose ps` to inspect, `docker compose logs <service>` to debug, `docker compose down` to stop.
- Ensure host media/download paths exist and are writable; `/mnt/media-1tb` defaults can be overridden in `.env`.
- GPU note: Jellyfin uses `runtime: nvidia`; adjust/remove if GPU unavailable.

## Config notes
- `.env` holds secrets and paths; `.env.example` shows required keys.
- Service configs live under `configs/...` at runtime and contain API keys and auth; keep them out of git and rotate if leaked.
- Uptime Kuma DB: `configs/uptime-kuma/kuma.db` (runtime-generated).

## Important endpoints (LAN)
- Jellyfin: `http://<host>:8096`
- Jellyseerr: `http://<host>:5055`
- Sonarr: `http://<host>:8989`
- Radarr: `http://<host>:7878`
- Prowlarr: `http://<host>:9696`
- Transmission: `http://<host>:9091`
- Nginx Proxy Manager: `http://<host>:8181`
- Uptime Kuma: `http://<host>:3001`
