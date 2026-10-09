# 5. Operations: Updates, Backups and Troubleshooting

[← Configure the apps](04-configure-apps.md) · [Overview](README.md)

## 5.1 Day-to-day commands

Run these from `~/docker/media-stack`:

```bash
docker compose ps                    # status
docker compose logs -f sonarr        # follow one app's logs
docker compose restart radarr        # restart one app
docker compose down                  # stop everything
docker compose up -d                 # start everything
docker stats --no-stream             # CPU / RAM per container
```

## 5.2 Updating

```bash
cd ~/docker/media-stack
docker compose pull                  # fetch new images
docker compose up -d                 # recreate only the changed containers
docker image prune -f                # remove old images (frees SSD space)
```

Do this every week or two, and **not** in the middle of an evening of watching. Read the release notes for major Jellyfin versions; they sometimes run long database migrations, which can take a while on a Pi.

Raspberry Pi OS itself:

```bash
sudo apt update && sudo apt full-upgrade -y && sudo reboot
```

> **Pinning versions:** to control upgrades, replace `:latest` with explicit tags (e.g. `jellyfin/jellyfin:10.11.x`, `lscr.io/linuxserver/sonarr:4.x.x`) and change them on purpose.

## 5.3 Backups

| What | Where | How |
|---|---|---|
| App config + databases | `~/docker/media-stack/config/` | The important part. Back it up. |
| Compose + secrets | `~/docker/media-stack/compose.yaml`, `.env` | Small. Back them up too. |
| Jellyfin cache | `~/docker/media-stack/cache/` | Not needed (it's rebuilt automatically) |
| Media | `/mnt/data/media` | Your own choice (NAS snapshots, external disk). It can be re-downloaded. |

Sonarr, Radarr and Prowlarr also create **automatic weekly backups** in `config/<app>/Backups/`. You can restore one in the app under *System → Backup*.

Simple consistent backup to the NAS or data disk:

```bash
cd ~/docker/media-stack
docker compose stop
sudo tar --exclude='./config/jellyfin/log' --exclude='./cache' \
  -czf "/mnt/data/backups/media-stack-$(date +%F).tar.gz" .
docker compose start
```

(Create `/mnt/data/backups` first.) You can schedule this weekly with `sudo crontab -e`.

**Restore on a fresh Pi:** follow chapters 1–2, extract the archive to `~/docker/media-stack`, then run `docker compose up -d`.

## 5.4 Troubleshooting

### Permission denied / import failed

- Check ownership: `ls -ln /mnt/data/media /mnt/data/torrents`. Files should be owned by your `PUID:PGID`.
- Make sure every service in `compose.yaml` uses the same `PUID`/`PGID` (`docker compose config | grep -E 'PUID|PGID|user:'`).
- NFS: the NAS export must allow writes for that UID (see [2.3 Option B](02-storage.md#option-b-nas-over-nfs-recommended-for-network-storage)).
- SMB: check the `uid=`, `gid=` and `file_mode`/`dir_mode` mount options.

### Imports are copies, not hardlinks (disk fills up twice as fast)

- Run the hardlink test from [2.5](02-storage.md#25-verify-that-hardlinks-work).
- Make sure Sonarr/Radarr mount the **whole** `${DATA_ROOT}:/data`, not separate `/downloads` and `/tv` volumes.
- Make sure qBittorrent's "Keep incomplete torrents in" isn't pointing to another disk.

### Sonarr/Radarr: "Unable to connect to qBittorrent"

- Host must be **`gluetun`**, not `qbittorrent` or `localhost`.
- Is the VPN up? `docker compose ps gluetun` should show `healthy`. If Gluetun is down, qBittorrent is unreachable by design.
- If the web UI is unreachable from the LAN or from other containers, add `- FIREWALL_INPUT_PORTS=8080` to Gluetun's environment and run `docker compose up -d`.

### "Path does not exist" in Sonarr/Radarr

The download client reports a path the app can't see. All containers must use the same `/data/...` paths (see the notes in [3.4](03-deploy-stack.md#notes-on-the-compose-file)). With this layout you never need *Remote Path Mappings*. Remove any you added.

### Gluetun keeps restarting / VPN won't connect

```bash
docker logs gluetun --tail 50
```

- Wrong or expired WireGuard key: regenerate it with your provider.
- `SERVER_COUNTRIES` value not recognised: check the spelling in the Gluetun wiki for your provider.
- `/dev/net/tun` missing: `ls -l /dev/net/tun`. It should exist on Raspberry Pi OS. If not, run `sudo modprobe tun`.

### Jellyfin buffers or stutters

- *Dashboard → Activity*, or the playback info overlay on the client: if it says **Transcoding**, find out why (codec, subtitles, bitrate limit, audio format).
- Fix it on the client side (a better client app, max bitrate, SRT subtitles) or the source side (Sonarr/Radarr profiles: 1080p, no Remux/4K).
- Check temperatures: `vcgencmd measure_temp`. Sustained temperatures above ~80 °C cause throttling (`vcgencmd get_throttled` should return `0x0`).

### Disk unexpectedly full / SD card full

The storage probably wasn't mounted when the containers started, so they wrote into the empty `/mnt/data` on the root filesystem.

```bash
findmnt /mnt/data                     # nothing printed = not mounted
sudo du -xh --max-depth=2 /mnt/data   # data on the root fs?
```

Stop the stack, move or remove the stray files, mount the storage, and make sure [2.6](02-storage.md#26-dont-start-docker-without-the-storage) is applied.

### Pi slow or out of memory

```bash
free -h
docker stats --no-stream
```

- Library scans in Jellyfin and big imports cause temporary spikes. A 4 GB Pi may need a larger swap (`sudo dphys-swapfile` / zram settings, depending on the OS release) or fewer simultaneous tasks.
- Reduce qBittorrent's max active downloads/torrents (*Options → BitTorrent → Torrent Queueing*).

## 5.5 Where to go next (out of scope here)

- Reverse proxy with HTTPS (nginx, Caddy, Traefik) and nice hostnames
- Secure remote access (Tailscale / WireGuard)
- Request portal for family members (e.g. Jellyseerr)
- Usenet, music (Lidarr) and books
- TRaSH Guides custom formats, synced with Recyclarr
