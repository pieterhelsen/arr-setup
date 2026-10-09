# 4. Configure and Connect the Apps

[← Deploy the stack](03-deploy-stack.md) · [Next: Operations →](05-operations.md)

Configure the apps in this order, because each step depends on the previous one:

```
qBittorrent → Prowlarr (+FlareSolverr) → Sonarr / Radarr → Prowlarr apps sync → Bazarr → Jellyfin
```

**Hostnames to use between containers:**

| From an app, connect to… | Host | Port |
|---|---|---|
| qBittorrent | `gluetun` ⚠️ | 8080 |
| Prowlarr | `prowlarr` | 9696 |
| Sonarr | `sonarr` | 8989 |
| Radarr | `radarr` | 7878 |
| FlareSolverr | `flaresolverr` | 8191 |
| Jellyfin | `jellyfin` | 8096 |

**API keys:** In Sonarr, Radarr and Prowlarr the key is under **Settings → General → Security → API Key**. In Bazarr it's under **Settings → General**.

---

## 4.1 qBittorrent

Open `http://<pi-ip>:8080` and log in as `admin` with the temporary password from [3.5](03-deploy-stack.md#get-the-qbittorrent-temporary-password).

**Tools → Options:**

| Section | Setting | Value |
|---|---|---|
| Web UI → Authentication | Username / Password | Set your own. The temporary password changes on every restart until you do. |
| Downloads → Saving Management | Default Torrent Management Mode | **Automatic** |
| | When Torrent Category changed | **Relocate torrent** |
| | When Default/Category Save Path changed | **Relocate affected torrents** |
| Downloads | Default Save Path | `/data/torrents` |
| Downloads | Keep incomplete torrents in | *unchecked* (keeps everything on one filesystem) |
| Downloads | Pre-allocate disk space for all files | Optional. Reduces fragmentation on HDDs. |
| BitTorrent → Seeding Limits | When ratio reaches / seeding time | Your choice, e.g. ratio 2 **then Pause torrent**. Use *Pause*, not *Remove*, and let Sonarr/Radarr remove them. |
| Connection | Peer connection protocol | TCP and µTP |
| Speed | Global limits | Set upload to ~80% of your line speed if streaming suffers |

Create the **categories** (left sidebar → *Categories* → right-click → *Add category*):

| Category | Save path |
|---|---|
| `tv` | `/data/torrents/tv` |
| `movies` | `/data/torrents/movies` |

> **Port forwarding:** Many VPN providers (e.g. NordVPN) don't support incoming port forwarding. Downloads still work, but you connect to fewer peers and seed less. Providers that support it (e.g. ProtonVPN, PIA, AirVPN) can be integrated through Gluetun's `VPN_PORT_FORWARDING` options. That setup is provider-specific; see the Gluetun wiki.

---

## 4.2 Prowlarr + FlareSolverr

Open `http://<pi-ip>:9696`. On first launch, set up **authentication** (Forms login, and *Authentication Required: Disabled for Local Addresses* if you like).

1. **FlareSolverr proxy:** *Settings → Indexers → `+` → FlareSolverr*
   - Tags: `flaresolverr`
   - Host: `http://flaresolverr:8191/`
   - Test → Save
2. **Indexers:** *Indexers → Add Indexer*. Search for your trackers and add them.
   - If an indexer fails with a Cloudflare error, edit it and add the tag `flaresolverr`.
3. Apps (Sonarr/Radarr) are added in [4.5](#45-connect-prowlarr-to-sonarr-and-radarr), after those apps are configured.

---

## 4.3 Sonarr (TV)

Open `http://<pi-ip>:8989` and set up authentication on first launch.

**Settings → Media Management** (click *Show Advanced*):

| Setting | Value |
|---|---|
| Rename Episodes | ✅ |
| Use Hardlinks instead of Copy | ✅ (default) |
| Unmonitor Deleted Episodes | ✅ |
| Set Permissions | ❌ (UMASK handles this) |
| **Root Folders → Add Root Folder** | `/data/media/tv` |

**Settings → Download Clients → `+` → qBittorrent:**

| Field | Value |
|---|---|
| Host | **`gluetun`** |
| Port | `8080` |
| Username / Password | from 4.1 |
| Category | `tv` |
| Remove Completed | ✅ (removes the torrent after seeding limits are reached; the hardlinked library file stays) |

Test → Save.

**Settings → Profiles (important on a Pi):**

Edit the profile you'll use (e.g. *HD-1080p*) so that it only allows qualities your clients can **direct play**:

- ✅ WEBDL-1080p, WEBRip-1080p, Bluray-1080p (and 720p equivalents)
- ❌ Remux and 2160p/4K. These are huge files that will need transcoding on many clients, and the Pi can't transcode them.

> For finer control (preferring x264 over x265, avoiding HDR/DV, release group scoring), have a look at **Custom Formats** in the [TRaSH Guides](https://trash-guides.info/), optionally synced automatically with Recyclarr. This is optional and not needed to get started.

---

## 4.4 Radarr (Movies)

Open `http://<pi-ip>:7878`. The setup is the same as Sonarr, with these differences:

| Setting | Value |
|---|---|
| Media Management → Rename Movies | ✅ |
| Media Management → Use Hardlinks instead of Copy | ✅ |
| Root Folder | `/data/media/movies` |
| Download Client → qBittorrent | Host **`gluetun`**, port `8080`, Category **`movies`** |
| Profiles | Same advice: cap at 1080p, no Remux/4K |

---

## 4.5 Connect Prowlarr to Sonarr and Radarr

Back in Prowlarr: *Settings → Apps → `+`*:

| Field | Sonarr | Radarr |
|---|---|---|
| Sync Level | Full Sync | Full Sync |
| Prowlarr Server | `http://prowlarr:9696` | `http://prowlarr:9696` |
| Sonarr/Radarr Server | `http://sonarr:8989` | `http://radarr:7878` |
| API Key | Sonarr's API key | Radarr's API key |

Test → Save. Then click **Sync App Indexers** (*System → Tasks* or the Indexers page). In Sonarr/Radarr, *Settings → Indexers* should now list your indexers.

---

## 4.6 Bazarr (Subtitles)

Open `http://<pi-ip>:6767`.

1. **Settings → General → Security:** enable authentication.
2. **Settings → Languages:**
   - Add languages (e.g. English, Dutch)
   - Create a **Languages Profile** and set it as the default for series and movies
3. **Settings → Providers:** add providers, e.g. *OpenSubtitles.com* (free account needed). Add a few, because one provider rarely covers everything.
4. **Settings → Sonarr:** enable, Address `sonarr`, Port `8989`, API key → Test → Save.
5. **Settings → Radarr:** enable, Address `radarr`, Port `7878`, API key → Test → Save.

No path mappings are needed, because Bazarr sees `/data/media` exactly as Sonarr/Radarr do.

---

## 4.7 Jellyfin

Open `http://<pi-ip>:8096` and complete the setup wizard:

1. Language, then create your **admin user**.
2. **Add media libraries:**

   | Content type | Folder |
   |---|---|
   | Shows | `/data/media/tv` |
   | Movies | `/data/media/movies` |

   Leave the metadata language/country at your preference. Real-time monitoring can stay on (it works for local disks).
3. Finish the wizard and log in.

### Playback settings for a Pi

*Dashboard → Playback → Transcoding:*

| Setting | Value |
|---|---|
| Hardware acceleration | **None** (no supported hardware encoder on the Pi) |
| Encoding thread count | Auto |
| Throttle Transcodes | ✅ |

*Dashboard → Users → \<user\> → Playback* for users on weak or remote clients: consider unchecking **"Allow video playback that requires transcoding"**. They then get an error instead of a stuttering stream, which makes the problem obvious.

On **clients**, set the max streaming bitrate to the maximum / auto so they don't force transcoding unnecessarily.

> **Subtitles:** "Burned-in" (image-based PGS/VobSub) subtitles force a transcode. Text subtitles (SRT, which Bazarr downloads) don't. Prefer SRT.

### Let Sonarr/Radarr notify Jellyfin

This makes new episodes and movies appear immediately. It's **required** when using NFS/SMB storage, where real-time monitoring doesn't work.

1. Jellyfin: *Dashboard → API Keys → `+`* → name it `sonarr-radarr`, then copy the key.
2. Sonarr **and** Radarr: *Settings → Connect → `+` → Emby / Jellyfin*:
   - Host: `jellyfin`, Port: `8096`
   - API Key: the key from step 1
   - ✅ Update Library
   - Triggers: On Import, On Upgrade, On Rename, On Delete
   - Test → Save

---

## 4.8 End-to-end test

1. In Radarr: *Movies → Add New*, search for a (legally downloadable / public-domain) movie, choose root folder `/data/media/movies` and your 1080p profile, and click **Add + Search**.
2. *Activity → Queue* shows it grabbing. qBittorrent shows it downloading under category `movies`.
3. When it finishes, Radarr imports it. Check that it's a **hardlink**:

   ```bash
   find /mnt/data/media/movies -type f -name '*.mkv' -exec stat -c 'links=%h  %n' {} \; | head
   # links=2 → hardlinked (also present in /mnt/data/torrents/movies)
   ```

4. The movie appears in Jellyfin within a few seconds, and Bazarr fetches subtitles shortly after.

## Checklist

- [ ] qBittorrent password changed, categories `tv` and `movies` created
- [ ] Prowlarr has indexers and syncs them to Sonarr + Radarr
- [ ] Sonarr/Radarr: root folders under `/data/media`, qBittorrent via host `gluetun`
- [ ] Bazarr connected to Sonarr + Radarr, with at least one provider
- [ ] Jellyfin libraries point to `/data/media/{tv,movies}` and receive Connect notifications
- [ ] Test download imports as a hardlink and plays in Jellyfin

[Next: Operations →](05-operations.md)
