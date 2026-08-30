# Homelab services

Self-hosted services running on the Raspberry Pi, reachable **only through the VPN**.
Nothing is exposed to the internet.

This document follows the same convention as [SETUP.md](SETUP.md): every command is
listed with an explanation of what it does and why.

## Design decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Where services live | Raspberry Pi (home) | Sensitive data never leaves the house; not even the VPS provider can reach it |
| How they are reached | WireGuard only | No ports opened, no public exposure |
| Deployment | Docker + Compose | Reproducible, isolated, easy to back up |
| Persistent data | External USB disk | microSD wears out fast with constant database writes |

## Hardware

- Raspberry Pi 5, 16 GB RAM (~15 GB usable).
- Boot: microSD (58 GB).
- Storage: external USB disk (500 GB) mounted at `/mnt/storage`.

---

## 1. Storage — external USB disk

The Pi boots from microSD, but databases and container volumes must **not** live there:
flash cards are slow and wear out quickly under the constant small writes that databases
produce. Everything persistent goes to the external disk.

### 1.1 Identify the disk

```bash
lsblk -f
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS /dev/sda
```

Check `TRAN` reads **`usb`** and that the size and model match the intended disk. The
microSD appears as `mmcblk0`, so it cannot be confused with `sda` — but verifying before
a destructive operation is worth the extra second.

### 1.2 Format as ext4

```bash
sudo umount /dev/sda1 2>/dev/null
sudo mkfs.ext4 -L homelab /dev/sda1
```

- **ext4** rather than NTFS/exFAT: it supports POSIX ownership and permissions, which
  Docker volumes and databases require. A disk previously used on Windows will be NTFS
  and must be reformatted.
- `-L homelab` sets a human-readable label.

### 1.3 Get the UUID

```bash
sudo blkid /dev/sda1
```

### 1.4 Mount permanently

```bash
sudo mkdir -p /mnt/storage
sudo cp /etc/fstab /etc/fstab.bak      # always back up before editing fstab
sudo nano /etc/fstab
```

Line to add:
```
UUID=<DISK_UUID>  /mnt/storage  ext4  defaults,noatime,nofail  0  2
```

Why each part matters:

- **`UUID=` instead of `/dev/sda1`** — device names are assigned in detection order and
  can change between boots (especially if another USB device is attached). A stale
  `fstab` entry would fail the boot or mount the wrong disk. The UUID travels with the
  filesystem.
- **`noatime`** — stops updating the "last access" timestamp on every read. Fewer writes,
  better performance.
- **`nofail`** — critical: the system still boots if the disk is missing. Without it, an
  unplugged or failing disk leaves the Pi hanging at boot.
- **`0 2`** — no `dump` backups; `fsck` order 2 (order 1 is reserved for root).

Apply and verify:
```bash
sudo systemctl daemon-reload      # systemd caches fstab; reload after editing
sudo mount -a
df -h /mnt/storage
```

### 1.5 Directory layout

```bash
sudo mkdir -p /mnt/storage/docker /mnt/storage/appdata
sudo chown -R $USER:$USER /mnt/storage/appdata
```

- `/mnt/storage/docker` — Docker's `data-root` (images, volumes, containers).
- `/mnt/storage/appdata` — per-service persistent configuration.

---

## 2. Docker

### 2.1 Install

```bash
curl -fsSL https://get.docker.com | sh
```
The official convenience script detects the distribution and architecture (Debian /
arm64 here), adds Docker's repository and installs `docker-ce`, `containerd` and the
`docker compose` plugin.

### 2.2 Run Docker without sudo

```bash
sudo usermod -aG docker $USER
```

Group membership is evaluated **at login**, so the current SSH session will still fail
with:
```
permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```
Log out and back in (or run `newgrp docker`) for it to take effect. Verify with `groups`
— `docker` must be listed.

> Security note: membership of the `docker` group is equivalent to root, since a
> container can bind-mount any host path. Acceptable on a personal server, worth knowing.

### 2.3 Move Docker's data directory to the external disk

```bash
sudo systemctl stop docker.socket docker.service
sudo nano /etc/docker/daemon.json
```
```json
{
  "data-root": "/mnt/storage/docker"
}
```
```bash
sudo systemctl start docker
docker info | grep "Docker Root Dir"     # must print /mnt/storage/docker
```

Stop **both** `docker.service` and `docker.socket`. Stopping only the service leaves the
socket active, which can re-launch the daemon on the next API call and reload the old
configuration. The warning looks like this:
```
Stopping 'docker.service', but its triggering units are still active: docker.socket
```

### 2.4 Verify

```bash
docker --version
docker compose version
docker run --rm hello-world
```
`--rm` removes the container after it exits, leaving nothing behind.

Two harmless warnings appear on Raspberry Pi OS:
```
WARNING: No memory limit support
WARNING: No swap limit support
```
They mean per-container memory limits are unavailable because the cgroup memory
controller is not enabled in the kernel command line. To enable it, append
`cgroup_enable=memory cgroup_memory=1` to `/boot/firmware/cmdline.txt` and reboot. Only
needed when limiting a heavy service.

---

## 3. Service layout

```
~/homelab/
└── <service>/
    ├── docker-compose.yml
    └── .env               # secrets — never committed
```

Persistent data is bind-mounted from `/mnt/storage/appdata/<service>`, so a service can
be recreated from scratch without losing its state, and backups only need to cover that
one directory tree.

---

## Status

- [x] External disk formatted and mounted at `/mnt/storage` (`nofail`, by UUID).
- [x] Docker + Compose installed, `data-root` on the external disk.
- [ ] Pi-hole + Unbound (DNS with ad blocking and recursive resolution).
- [ ] Reverse proxy (Caddy) with internal HTTPS.
- [ ] Vaultwarden (password manager).
- [ ] Backups (restic to the VPS).
- [ ] Uptime Kuma (monitoring and alerts).

## Planned: Pi-hole + Unbound

The next service, and the one with the best synergy with the existing VPN.

**Why it fits.** Setting `DNS = 10.10.0.2` in the WireGuard client profiles means DNS
queries are resolved by the Pi **through the tunnel**, so ad blocking works on mobile
data away from home, with no app installed. It also removes the dependency on
`1.1.1.1` currently set in the full-tunnel profile, closing the "own DNS resolver" item
from Phase 5.

**Why Unbound as well.** Pi-hole only filters; it forwards everything else upstream. With
Unbound acting as a recursive resolver, queries are resolved directly against the root
servers, so no third party sees the browsing history:

```
client → Pi-hole (blocked?) → Unbound → root DNS servers
              ↓ yes
         0.0.0.0 (blocked)
```

**Prerequisite already verified.** Pi-hole needs UDP/TCP port 53. On this system it is
free — `systemd-resolved` is not present on Raspberry Pi OS, and `avahi-daemon` uses
5353 (mDNS), a different port:
```bash
sudo ss -tulnp | grep :53
systemctl is-active systemd-resolved     # inactive / not-found
```

**Rollout order.** VPN clients first, then optionally the whole house via the router.
Pointing the router at Pi-hole filters every device (TVs, IoT) but makes the Pi a single
point of failure for DNS, so a secondary resolver should be configured there first.

**Security constraint.** Pi-hole must listen only on the LAN and VPN interfaces, never
publicly: an open DNS resolver is abused for DNS amplification DDoS attacks.
