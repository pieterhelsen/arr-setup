# 2. Storage Strategy

[← Prerequisites](01-prerequisites.md) · [Next: Deploy the stack →](03-deploy-stack.md)

Storage is where most *arr setups go wrong. This chapter gives one generic strategy that works whether your media is on a USB disk attached to the Pi, on a NAS over NFS, or on a Samba/SMB share.

## 2.1 The rules

1. **One data root, one filesystem.** Downloads and the media library live under a single directory (`/mnt/data` on the Pi) on **the same filesystem / the same network share**. This is what makes **hardlinks** and **instant (atomic) moves** possible:
   - Sonarr/Radarr "import" a finished torrent by creating a hardlink in the library. This takes no time and no extra space.
   - qBittorrent keeps seeding the original path.
   - When you later delete the torrent, the library copy remains.

   If downloads and media are on different disks or shares, every import becomes a full **copy**, which is slow on a Pi and uses double the space.

2. **Same paths everywhere.** Every container sees the data at **`/data`**. Paths reported by qBittorrent then mean the same thing to Sonarr, Radarr, Bazarr and Jellyfin, so you never need "remote path mappings".

3. **App config stays local.** `/config` folders (SQLite databases) live on the Pi's own SSD/SD card (`~/docker/media-stack/config`), **never** on NFS/SMB.

4. **One owner.** All files are owned by your `PUID:PGID` (from [1.5](01-prerequisites.md#15-collect-your-ids)), with group-writable permissions (`UMASK=002`).

## 2.2 Folder layout

This follows the [TRaSH Guides](https://trash-guides.info/File-and-Folder-Structure/) layout:

```
/mnt/data                 ← host path (local disk, NFS or SMB mount)
├── torrents              ← qBittorrent downloads here
│   ├── movies            ← category "movies" (Radarr)
│   └── tv                ← category "tv" (Sonarr)
└── media                 ← Jellyfin libraries
    ├── movies            ← Radarr root folder
    └── tv                ← Sonarr root folder
```

Inside the containers this appears as `/data/torrents/...` and `/data/media/...`.

## 2.3 Choose a storage option

| Option | When to use | Hardlinks | Performance on a Pi |
|---|---|---|---|
| **A. Local USB/NVMe disk** | Simplest. Only the Pi uses the media. | ✅ Yes | Best |
| **B. NAS over NFS** | You already have a NAS (Synology, TrueNAS, Unraid, OMV, Linux server) | ✅ Yes, if `torrents/` and `media/` are in the **same export** | Good (Gigabit) |
| **C. NAS / Windows over SMB (Samba)** | The NAS only offers SMB, or the share is on Windows | ⚠️ Depends on the server, so **test it** (2.5) | Good, with slightly more CPU overhead |

Prefer **NFS over SMB** when the NAS supports both. Linux handles NFS ownership and permissions natively, and hardlinks work reliably.

Pick **one** option below. Each ends with the data root mounted at **`/mnt/data`**.

---

### Option A: Local USB or NVMe disk

> **Power:** a 2.5" USB HDD can draw more than the Pi's USB ports supply. Use the 27 W PSU and/or a powered USB hub. 3.5" drives need an enclosure with its own power supply.

1. Identify the disk (**double-check this, the next step erases it**):

   ```bash
   lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINT
   ```

2. Partition and format it as **ext4** (skip this if it already has an ext4 filesystem you want to keep). Example for `/dev/sda`:

   ```bash
   sudo parted /dev/sda --script mklabel gpt mkpart data ext4 0% 100%
   sudo mkfs.ext4 -L data /dev/sda1
   ```

   > Avoid NTFS/exFAT. They don't support Linux permissions or hardlinks properly.

3. Mount it permanently, by UUID:

   ```bash
   sudo blkid /dev/sda1                 # copy the UUID="..."
   sudo mkdir -p /mnt/data
   echo 'UUID=<your-uuid>  /mnt/data  ext4  defaults,noatime,nofail,x-systemd.device-timeout=30  0  2' | sudo tee -a /etc/fstab
   sudo systemctl daemon-reload
   sudo mount -a
   df -h /mnt/data
   ```

   `nofail` lets the Pi still boot if the disk is unplugged. Section 2.6 makes sure Docker doesn't start without it.

> **Several disks?** Keep it simple: use one large disk for `/mnt/data`. Pooling disks (mergerfs, RAID, LVM) is possible but out of scope. With mergerfs, hardlinks only work when source and target end up on the same underlying disk.

---

### Option B: NAS over NFS (recommended for network storage)

**On the NAS** (the exact menus differ per vendor):

1. Create **one** shared folder, e.g. `data`, and inside it the folders from 2.2 (or let the Pi create them in 2.4).
2. Enable **NFS** (v4.1 if offered).
3. Add an NFS permission/export rule for the **Pi's IP** with **read/write**.
4. Make sure the Pi's `PUID:PGID` can write to the share. Two common ways:
   - **Matching IDs:** the folder is owned by a NAS user/group with the same UID/GID as the Pi user (e.g. 1000:1000), with squash set to *no mapping* (`no_root_squash` is **not** required).
   - **Squash all to one user:** map all clients to the NAS user that owns the share (Synology: *Squash → Map all users to admin/guest*; TrueNAS: *Mapall User/Group*; Linux: `all_squash,anonuid=<uid>,anongid=<gid>`).

   Example `/etc/exports` on a plain Linux NAS:

   ```
   /srv/data  192.168.1.50(rw,sync,no_subtree_check)
   ```

| NAS | Where | Typical export path |
|---|---|---|
| Synology DSM | Control Panel → File Services → NFS (enable), then Shared Folder → Edit → NFS Permissions | `/volume1/data` |
| TrueNAS | Shares → Unix (NFS) Shares | `/mnt/<pool>/data` |
| OpenMediaVault | Services → NFS → Shares | `/export/data` |
| Unraid | Shares → data → NFS Security | `/mnt/user/data` |

**On the Pi:**

```bash
sudo apt install -y nfs-common
showmount -e <nas-ip>                    # should list your export
sudo mkdir -p /mnt/data
echo '<nas-ip>:/volume1/data  /mnt/data  nfs  defaults,_netdev,nofail,noatime,x-systemd.mount-timeout=30  0  0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/data
touch /mnt/data/.write-test && rm /mnt/data/.write-test && echo "write OK"
```

`_netdev` makes systemd wait for the network before mounting.

> **Jellyfin note:** Jellyfin's *real-time monitoring* relies on inotify, which doesn't fire for changes made by other machines on network shares. Section 4 adds a Sonarr/Radarr → Jellyfin "Connect" notification so new media still appears immediately.

---

### Option C: SMB / Samba share

Use this when the NAS (or a Windows machine) only offers SMB.

**On the NAS:** create one share (e.g. `data`) and a dedicated user (e.g. `mediapi`) with read/write access.

**On the Pi:**

```bash
sudo apt install -y cifs-utils

# Credentials file, readable by root only
sudo tee /root/.smb-data >/dev/null <<'EOF'
username=mediapi
password=<share-password>
EOF
sudo chmod 600 /root/.smb-data

sudo mkdir -p /mnt/data
# Replace uid/gid with your PUID/PGID
echo '//<nas-ip>/data  /mnt/data  cifs  credentials=/root/.smb-data,uid=1000,gid=1000,file_mode=0664,dir_mode=0775,vers=3.1.1,iocharset=utf8,_netdev,nofail,x-systemd.mount-timeout=30  0  0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/data
```

- `uid`/`gid`/`file_mode`/`dir_mode`: SMB doesn't carry Linux ownership, so the mount presents every file as owned by your `PUID:PGID`. The containers can then write.
- `vers=3.1.1`: lower it to `3.0` for older NAS firmware. Avoid `vers=1.0`.
- Then run the **hardlink test in 2.5**. If it fails, imports are copies rather than hardlinks. That still works, but it is slower and uses double space while seeding. In that case, consider NFS or a local disk.

---

## 2.4 Create the folders and set ownership

Run this once, whichever option you chose (replace `1000:1000` with your `PUID:PGID`):

```bash
sudo mkdir -p /mnt/data/{torrents/{movies,tv},media/{movies,tv}}
sudo chown -R 1000:1000 /mnt/data/torrents /mnt/data/media
sudo chmod -R u=rwX,g=rwX,o=rX /mnt/data/torrents /mnt/data/media
tree -d -L 2 /mnt/data 2>/dev/null || find /mnt/data -maxdepth 2 -type d
```

> On SMB the `chown`/`chmod` calls may fail or be ignored. That's expected, because the mount options control ownership.

## 2.5 Verify that hardlinks work

```bash
echo test > /mnt/data/torrents/hl-test
ln /mnt/data/torrents/hl-test /mnt/data/media/hl-test && \
  stat -c 'links=%h  %n' /mnt/data/media/hl-test     # expect: links=2
rm -f /mnt/data/torrents/hl-test /mnt/data/media/hl-test
```

- `links=2` means hardlinks work. 🎉
- `Operation not permitted` / `Invalid cross-device link` means no hardlinks. Check that `torrents/` and `media/` really are on the same filesystem/share.

## 2.6 Don't start Docker without the storage

If `/mnt/data` fails to mount (disk unplugged, NAS offline), the empty `/mnt/data` directory is still there. The containers would then happily download into your **SD card/SSD** and fill it up, and Jellyfin would show empty libraries. Make Docker depend on the mount:

```bash
sudo systemctl edit docker.service
```

Add the following between the comment markers:

```ini
[Unit]
RequiresMountsFor=/mnt/data
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Trade-off: if the storage is missing at boot, Docker won't start. That is intended on a dedicated media Pi. Fix the storage, then run `sudo systemctl start docker`.

## 2.7 Optional: share the Pi's local disk over Samba

If you used **Option A** and want to drop files onto the disk from your PC, you can export it with Samba:

```bash
sudo apt install -y samba
sudo smbpasswd -a "$USER"            # set an SMB password for your Pi user
sudo tee -a /etc/samba/smb.conf >/dev/null <<EOF

[data]
   path = /mnt/data
   valid users = $USER
   read only = no
   create mask = 0664
   directory mask = 0775
EOF
sudo systemctl restart smbd
```

Then connect from your PC to `\\<pi-ip>\data` (Windows) or `smb://<pi-ip>/data` (macOS/Linux).

## Checklist

- [ ] `/mnt/data` is mounted and survives a reboot (`sudo reboot`, then `df -h /mnt/data`)
- [ ] `/mnt/data/{torrents,media}/{movies,tv}` exist and are writable by `PUID:PGID`
- [ ] Hardlink test prints `links=2` (or you have accepted copy-mode)
- [ ] Docker depends on `/mnt/data`

[Next: Deploy the stack →](03-deploy-stack.md)
