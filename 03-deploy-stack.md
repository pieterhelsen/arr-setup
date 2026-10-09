# 3. Deploy the Stack with Docker Compose

[← Storage](02-storage.md) · [Next: Configure the apps →](04-configure-apps.md)

All services are defined in a single Compose project in `~/docker/media-stack`:

```
~/docker/media-stack/
├── compose.yaml        ← service definitions
├── .env                ← your IDs, paths and VPN secrets (chmod 600)
└── config/             ← per-app config and databases (local SSD!)
    ├── gluetun/
    ├── qbittorrent/
    ├── prowlarr/
    ├── sonarr/
    ├── radarr/
    ├── bazarr/
    └── jellyfin/
```

## 3.1 Create the directories

```bash
mkdir -p ~/docker/media-stack/config/{gluetun,qbittorrent,prowlarr,sonarr,radarr,bazarr,jellyfin} \
         ~/docker/media-stack/cache/jellyfin
cd ~/docker/media-stack
```

Create the folders yourself rather than letting Docker do it. Docker creates missing bind-mount folders as `root`, and the Jellyfin container (which runs as your user) then can't write to them.

## 3.2 The VPN (Gluetun)

qBittorrent runs **inside Gluetun's network namespace**. All its traffic goes through the VPN tunnel, and if the tunnel is down it has no connectivity at all, which works as a kill switch.

Gluetun supports most providers (NordVPN, Mullvad, ProtonVPN, Surfshark, PIA, AirVPN, custom WireGuard/OpenVPN…). You need:

- `VPN_SERVICE_PROVIDER`: provider name as listed in the [Gluetun wiki](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
- a **WireGuard private key** (or OpenVPN username/password). How you get it depends on the provider and is described on the provider's wiki page. WireGuard is faster and lighter on a Pi.

> Don't want a VPN? See [3.6 Running without a VPN](#36-running-without-a-vpn).

## 3.3 `.env`

Create `~/docker/media-stack/.env`:

```dotenv
# --- Identity (from `id`, see 1.5) ---
PUID=1000
PGID=1000
TZ=Europe/Brussels

# --- Storage (see chapter 2) ---
DATA_ROOT=/mnt/data

# --- Network ---
# Your home LAN; lets qBittorrent's web UI answer LAN clients through the VPN firewall
LAN_SUBNET=192.168.1.0/24

# --- VPN (Gluetun) ---
VPN_SERVICE_PROVIDER=nordvpn
VPN_SERVER_COUNTRIES=Belgium
WIREGUARD_PRIVATE_KEY=replace-me
```

Protect it, because it contains your VPN key:

```bash
chmod 600 .env
```

## 3.4 `compose.yaml`

Create `~/docker/media-stack/compose.yaml`:

```yaml
name: media-stack

services:

  # ───────────────────────── VPN + downloader ─────────────────────────
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - TZ=${TZ}
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - SERVER_COUNTRIES=${VPN_SERVER_COUNTRIES}
      - FIREWALL_OUTBOUND_SUBNETS=${LAN_SUBNET}
    volumes:
      - ./config/gluetun:/gluetun
    ports:
      - "8080:8080"          # qBittorrent web UI (published here, not on qbittorrent)
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"   # all traffic goes through the VPN
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=002
      - WEBUI_PORT=8080
      # Optional: nicer, mobile-friendly web UI
      # - DOCKER_MODS=ghcr.io/vuetorrent/vuetorrent-lsio-mod:latest
    volumes:
      - ./config/qbittorrent:/config
      - ${DATA_ROOT}/torrents:/data/torrents
    restart: unless-stopped

  # ───────────────────────── Indexers ─────────────────────────
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/prowlarr:/config
    ports:
      - "9696:9696"
    restart: unless-stopped

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - TZ=${TZ}
      - LOG_LEVEL=info
    # No ports published: only Prowlarr talks to it, over the internal network
    restart: unless-stopped

  # ───────────────────────── Automation ─────────────────────────
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=002
    volumes:
      - ./config/sonarr:/config
      - ${DATA_ROOT}:/data            # whole tree → hardlinks work
    ports:
      - "8989:8989"
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=002
    volumes:
      - ./config/radarr:/config
      - ${DATA_ROOT}:/data            # whole tree → hardlinks work
    ports:
      - "7878:7878"
    restart: unless-stopped

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=002
    volumes:
      - ./config/bazarr:/config
      - ${DATA_ROOT}/media:/data/media
    ports:
      - "6767:6767"
    restart: unless-stopped

  # ───────────────────────── Media server ─────────────────────────
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: "${PUID}:${PGID}"
    environment:
      - TZ=${TZ}
    volumes:
      - ./config/jellyfin:/config
      - ./cache/jellyfin:/cache
      - ${DATA_ROOT}/media:/data/media:ro   # read-only: Jellyfin never needs to modify media
    ports:
      - "8096:8096"
      # - "7359:7359/udp"   # optional: client auto-discovery on the LAN
    restart: unless-stopped
```

### Notes on the compose file

- **Identical `/data` paths:** qBittorrent sees `/data/torrents`, Sonarr/Radarr see all of `/data`, and Bazarr/Jellyfin see `/data/media`. A path such as `/data/torrents/tv/Show.S01E01.mkv` means the same thing in every container.
- **Reaching qBittorrent:** because qBittorrent shares Gluetun's network, other containers connect to it at **`gluetun:8080`**, not `qbittorrent:8080`. You reach it from your browser at `http://<pi-ip>:8080`.
- **Service names as hostnames:** all services are on the default Compose network, so they reach each other by service name (`http://sonarr:8989`, `http://prowlarr:9696`, `http://flaresolverr:8191`, …).
- **`:latest` tags:** convenient, but an update can occasionally break things. For a more predictable setup, pin version tags and bump them deliberately (see [Operations](05-operations.md)).
- **All images are multi-arch** and pull `arm64` variants automatically on a 64-bit Pi OS.

## 3.5 Start it

```bash
cd ~/docker/media-stack
docker compose config --quiet && echo "compose file OK"
docker compose pull
docker compose up -d
docker compose ps
```

On a Pi the first start can take a few minutes. All containers should show `running`, and `gluetun` should show `healthy`.

### Verify the VPN

```bash
# Gluetun logs: look for "Wireguard setup is complete" and your public IP
docker logs gluetun 2>&1 | grep -iE "public ip|wireguard|error" | tail

# Public IP as seen from inside qBittorrent's network: must NOT be your home IP
docker run --rm --network=container:gluetun alpine:3.20 wget -qO- https://ipinfo.io/ip; echo
curl -s https://ipinfo.io/ip; echo                       # your home IP, for comparison
```

### Get the qBittorrent temporary password

The linuxserver image generates a temporary admin password on first start:

```bash
docker logs qbittorrent 2>&1 | grep -i password
```

You'll change it in [4.1](04-configure-apps.md#41-qbittorrent).

### Open the web UIs

| App | URL |
|---|---|
| qBittorrent | `http://<pi-ip>:8080` |
| Prowlarr | `http://<pi-ip>:9696` |
| Sonarr | `http://<pi-ip>:8989` |
| Radarr | `http://<pi-ip>:7878` |
| Bazarr | `http://<pi-ip>:6767` |
| Jellyfin | `http://<pi-ip>:8096` |

## 3.6 Running without a VPN

Not recommended for public torrent trackers. If you skip the VPN:

1. Remove the `gluetun` service.
2. Change the `qbittorrent` service:

   ```yaml
     qbittorrent:
       # remove: network_mode and depends_on
       ports:
         - "8080:8080"
         - "6881:6881"
         - "6881:6881/udp"
   ```

3. In the apps, connect to qBittorrent at host **`qbittorrent`** instead of `gluetun`.

## Checklist

- [ ] `docker compose ps` shows all services running, `gluetun` healthy
- [ ] VPN IP differs from your home IP
- [ ] All web UIs load from your PC

[Next: Configure the apps →](04-configure-apps.md)
