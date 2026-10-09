# *arr Media Stack on a Single Raspberry Pi

Step-by-step guide to running a complete, self-hosted media automation stack on one Raspberry Pi with Docker Compose.

| Component | Role | Port |
|---|---|---|
| **Jellyfin** | Media server: streams your library to TVs, phones, browsers | 8096 |
| **qBittorrent** | Torrent client (routed through the VPN container) | 8080 |
| **Gluetun** | VPN client container; qBittorrent's only route to the internet (kill switch) | — |
| **Prowlarr** | Indexer manager: feeds torrent indexers to Sonarr/Radarr | 9696 |
| **Sonarr** | TV show automation | 8989 |
| **Radarr** | Movie automation | 7878 |
| **Bazarr** | Subtitle automation | 6767 |
| **FlareSolverr** | Solves Cloudflare challenges for some indexers (internal only) | 8191 |

Everything is reached on the local network as `http://<pi-ip>:<port>`.

## How it fits together

```
                    ┌──────────── Raspberry Pi (Docker) ─────────────┐
                    │                                                │
  You ──► Sonarr / Radarr ──search──► Prowlarr ──► indexers           │
                    │   │                 └─(FlareSolverr)           │
                    │   └──send torrent──► qBittorrent ═══ Gluetun ══╪══► VPN ──► internet
                    │                          │                     │
                    │                 /data/torrents/{tv,movies}     │
                    │                          │                     │
                    │   Sonarr/Radarr hardlink/move into             │
                    │                 /data/media/{tv,movies}        │
                    │                          │                     │
                    │   Bazarr adds subtitles ─┤                     │
                    │                          ▼                     │
  TV / phone ◄──────┼──────────────────── Jellyfin                   │
                    └────────────────────────────────────────────────┘
                                   │
                         /mnt/data (USB disk, NFS or SMB share)
```

## Documents

Follow these in order:

1. [Prerequisites: hardware and Raspberry Pi OS setup](01-prerequisites.md)
2. [Storage strategy: local disk, NAS (NFS) or Samba share](02-storage.md)
3. [Deploying the stack with Docker Compose](03-deploy-stack.md)
4. [Configuring and connecting the apps](04-configure-apps.md)
5. [Operations: updates, backups and troubleshooting](05-operations.md)

## Scope

**In scope**

- Raspberry Pi OS (64-bit) installation and basic hardening
- Docker + Docker Compose installation
- A storage layout that supports hardlinks and atomic moves, on local or network storage
- Deploying and wiring up the services listed above
- LAN-only access (`http://<pi-ip>:<port>`)

**Out of scope**

- **Reverse proxy** (nginx, Traefik, Caddy), domain names and HTTPS/TLS certificates
- **Remote access** from outside your home network (port forwarding, Tailscale/WireGuard to home, Cloudflare tunnels)
- Single sign-on / central authentication (Authelia, Authentik)
- **Hardware transcoding** in Jellyfin. The Pi 5 has no hardware video encoder, and Jellyfin has deprecated Pi hardware acceleration. This guide is built around *direct play*.
- Usenet (SABnzbd/NZBGet), music/books (Lidarr/Readarr), request front-ends (e.g. Jellyseerr)
- Setting up the NAS itself (RAID, snapshots, NAS user management) beyond what the Pi needs
- Monitoring/alerting stacks and automatic container updates (e.g. Watchtower)
- Legal advice: only download content you have the right to download.

## Design decisions (and why)

These choices are based on a working x86 setup, adapted for a Pi and with two known issues fixed:

| Decision | Reason |
|---|---|
| One `/data` tree with `torrents/` and `media/` on the **same filesystem**, mounted identically in every container | Lets Sonarr/Radarr **hardlink** finished downloads into the library. That is instant, uses no extra space, and torrents keep seeding. The reference setup mounted `/downloads`, `/movies` and `/shows` as separate disks/volumes, which forces a slow copy + delete and doubles disk usage while seeding. |
| One `PUID`/`PGID` for **all** containers, set in `.env` | Avoids permission errors between apps. The reference setup mixed UID 998 (arr) and 1000 (Jellyfin). |
| qBittorrent runs with `network_mode: service:gluetun` | If the VPN drops, qBittorrent has no network at all, so nothing leaks. |
| App config (`/config`, SQLite databases) on the Pi's **local** SSD, never on NFS/SMB | SQLite over network filesystems leads to locked or corrupted databases. |
| OS on SSD/NVMe instead of an SD card (recommended) | The *arr databases write constantly and wear out SD cards. |
