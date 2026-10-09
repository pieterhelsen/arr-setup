# 1. Prerequisites: Hardware and Raspberry Pi OS

[← Overview](README.md) · [Next: Storage →](02-storage.md)

## 1.1 Hardware

| Item | Recommendation | Notes |
|---|---|---|
| Board | **Raspberry Pi 5, 8 GB** (Pi 4 with 4 GB+ also works) | The full stack idles at about 1.5–2.5 GB RAM. Jellyfin library scans and Sonarr/Radarr imports cause spikes. |
| Power supply | Official **27 W USB-C** PSU (Pi 5) | Needed to power USB drives reliably. A weak PSU causes disk disconnects and corruption. |
| Cooling | Active Cooler or a case with a fan | Library scans and software transcoding keep the CPU busy for long periods. |
| Boot/system disk | **NVMe SSD via M.2 HAT** or a **USB 3 SSD**; a high-endurance microSD card at minimum | This disk holds the OS, Docker images and all app databases. |
| Media storage | USB HDD/SSD **or** a NAS on the LAN | See [Storage](02-storage.md). |
| Network | **Wired Ethernet** | Wi-Fi works, but streaming and NAS traffic suffer. |

> **About transcoding:** The Pi has no usable hardware video encoder for Jellyfin (the Pi 5 has none at all, and Jellyfin has deprecated Pi 4 V4L2 support). Software transcoding of 1080p is marginal and 4K is not possible. Plan for **direct play**: use clients that natively support your files (Jellyfin apps for Android TV, Fire TV, iOS, webOS, Kodi) and prefer 1080p H.264/HEVC releases. Section 4 sets up the quality profiles for this.

## 1.2 Flash Raspberry Pi OS

1. Download and run **Raspberry Pi Imager** on your PC (<https://www.raspberrypi.com/software/>).
2. Choose:
   - **Device:** your Pi model
   - **OS:** *Raspberry Pi OS (other)* → **Raspberry Pi OS Lite (64-bit)**. You don't need a desktop, and it must be 64-bit: several images are arm64-only.
   - **Storage:** the SSD (in a USB adapter) or the microSD card
3. In the OS customisation step, set:
   - **Hostname**, e.g. `mediapi`
   - **Username and password** (this user's UID will be `1000`)
   - **Locale / time zone / keyboard**
   - **SSH: enabled**, preferably with **public-key authentication** (paste your `~/.ssh/id_ed25519.pub`)
   - Leave Wi-Fi empty if you use Ethernet
4. Write the image, insert or attach it to the Pi, and boot.

### Booting from NVMe/USB SSD (Pi 5 / Pi 4)

Recent Pi 5 bootloaders try SD → NVMe → USB automatically. If the Pi does not boot from the SSD:

```bash
# Temporarily boot from an SD card, then:
sudo apt update && sudo apt full-upgrade -y
sudo rpi-eeprom-update -a          # update the bootloader
sudo raspi-config                  # Advanced Options → Boot Order → NVMe/USB Boot
sudo reboot
```

## 1.3 First login and base configuration

```bash
ssh <user>@mediapi.local       # or ssh <user>@<pi-ip>
```

Update the system:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

### Give the Pi a fixed IP address

All the apps are reached by IP, so the address must not change.

**Recommended:** create a **DHCP reservation** for the Pi's MAC address in your router. This needs no changes on the Pi.

**Alternative:** a static IP on the Pi. Raspberry Pi OS uses NetworkManager:

```bash
nmcli con show                                   # find the wired connection name
CON="netplan-eth0"                               # ← replace with the name shown
sudo nmcli con mod "$CON" ipv4.method manual \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "192.168.1.1"
sudo nmcli con up "$CON"
```

Write down the Pi's IP and your **LAN subnet** (e.g. `192.168.1.0/24`). You need both later.

### Optional hardening

```bash
# Automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Only if you set up SSH key login: disable password login
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

## 1.4 Install Docker and Docker Compose

Use Docker's official convenience script. It detects Raspberry Pi OS (Debian, arm64) and installs Docker Engine together with the `docker compose` plugin:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
less get-docker.sh                  # read before running
sudo sh get-docker.sh

# Run docker without sudo (log out and back in afterwards)
sudo usermod -aG docker "$USER"
```

Verify:

```bash
docker run --rm hello-world
docker compose version
```

### Limit container log size

By default Docker container logs grow without limit, which wears out the disk. Create `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

```bash
sudo systemctl restart docker
```

## 1.5 Collect your IDs

Every container runs as the same user so that file permissions line up:

```bash
id
# uid=1000(pi) gid=1000(pi) groups=...
```

Write down the `uid` and `gid` (usually `1000`/`1000`). These are your `PUID`/`PGID`.

> If your media will live on a **NAS**, this UID/GID must also have read/write access on the NAS share. [Storage](02-storage.md) covers this.

## Checklist

- [ ] Pi boots Raspberry Pi OS Lite 64-bit, ideally from SSD
- [ ] Fixed IP known, LAN subnet known
- [ ] SSH access works
- [ ] `docker run hello-world` works without `sudo`
- [ ] `PUID`/`PGID` noted

[Next: Storage →](02-storage.md)
