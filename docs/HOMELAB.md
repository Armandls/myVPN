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

- `/mnt/storage/docker` — Docker's `data-root` (volumes, networks, build cache).
- `/mnt/storage/containerd` — containerd's image store (see 2.3).
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

### 2.3 Move the storage off the microSD

Two separate settings are needed on Docker 29.x, which is easy to get wrong.

**Docker's own data directory** (volumes, networks, build cache):
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

**containerd's image store.** Docker 29.x enables the containerd snapshotter by default,
visible in the daemon log:
```
msg="Starting daemon with containerd snapshotter integration enabled"
```
With it, **images are stored by containerd in `/var/lib/containerd`**, not under Docker's
`data-root`. Setting `data-root` alone leaves every image on the microSD — exactly what
this was meant to avoid. Check with:
```bash
sudo du -sh /var/lib/containerd
sudo du -sh /mnt/storage/docker
```

Move it:
```bash
sudo systemctl stop docker.socket docker.service containerd

sudo mkdir -p /mnt/storage/containerd
sudo rsync -aHAX --info=progress2 /var/lib/containerd/ /mnt/storage/containerd/
sudo du -sh /mnt/storage/containerd      # confirm the size matches before continuing
```
`-H` (hard links), `-A` (ACLs) and `-X` (extended attributes) are required: containerd
relies on them for image layers, and a plain copy would corrupt the store.

```bash
sudo tee /etc/containerd/config.toml > /dev/null <<'EOF'
version = 2
root = "/mnt/storage/containerd"
state = "/run/containerd"
EOF
```
`root` is the persistent store, which is what moves. `state` stays under `/run` (tmpfs):
it is meant to be volatile and does not wear the card.

Once verified, free the space on the card:
```bash
sudo mv /var/lib/containerd /var/lib/containerd.old
df -h /
```
Renaming rather than deleting leaves a way back; remove it after a few days of normal
operation.

### 2.4 Wait for the disk at boot

Both daemons store data on an external USB disk, which takes longer to appear than the
services take to start. If they start first, they initialise an empty store and every
container vanishes.

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/override.conf > /dev/null <<'EOF'
[Unit]
RequiresMountsFor=/mnt/storage
EOF

sudo mkdir -p /etc/systemd/system/containerd.service.d
sudo tee /etc/systemd/system/containerd.service.d/override.conf > /dev/null <<'EOF'
[Unit]
RequiresMountsFor=/mnt/storage
EOF

sudo systemctl daemon-reload
```

`RequiresMountsFor=` makes systemd derive the mount unit for that path and treat it as a
dependency, so neither daemon starts before the disk is mounted.

This must be verified with an actual reboot — see issue 13 in
[TROUBLESHOOTING.md](TROUBLESHOOTING.md).

### 2.5 Verify

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

## 4. Pi-hole + Unbound

DNS with ad blocking, plus recursive resolution so no third party sees the browsing
history. Templates: [../pihole/docker-compose.yml.example](../pihole/docker-compose.yml.example)
and [../pihole/.env.example](../pihole/.env.example).

### Why both

Pi-hole only **filters**: it checks a domain against blocklists and, if allowed, has to
ask someone else for the real address. By default that someone is a public resolver
(Cloudflare, Google), which then sees every domain visited.

Unbound is a **recursive resolver**: it answers by querying the DNS hierarchy itself,
starting at the root servers. Combining both gives ad blocking *and* DNS privacy.

```
client → Pi-hole (in a blocklist?) → Unbound → root servers → .com → google.com
              ↓ yes
         0.0.0.0 (blocked)
```

They run as two containers rather than one because each is a separate concern: they can
be updated independently, a crash in one does not take the other down, and both come
from images maintained upstream.

### Why the port 53 prerequisite matters

Pi-hole needs UDP/TCP port 53. Verified free on this system before starting:
```bash
sudo ss -tulnp | grep :53
systemctl is-active systemd-resolved     # inactive / not-found
```
Raspberry Pi OS does not ship `systemd-resolved`, and `avahi-daemon` uses 5353 (mDNS),
a different port. On distributions that do run `systemd-resolved`, its stub listener
must be disabled first.

### Network modes: different for each container

This is the subtle part. The two containers use **different** network modes on purpose:

- **Pi-hole → `network_mode: host`.** It shares the Pi's network stack, so it listens on
  every interface at once — LAN and the WireGuard interface — and sees the real client
  IPs in its statistics. With bridge mode it would report the Docker gateway as the
  client for every query.
- **Unbound → bridge with `127.0.0.1:5335:53`.** The `mvance/unbound-rpi` image listens
  on port 53 *inside* the container. In host mode it would compete with Pi-hole for the
  host's port 53 and neither would work. The explicit mapping moves it to 5335 on the
  host, bound to loopback only so it is unreachable from the network.

### Deploy

```bash
mkdir -p ~/homelab/pihole
mkdir -p /mnt/storage/appdata/pihole/etc-pihole
mkdir -p /mnt/storage/appdata/unbound
```

Ownership matters: create these as your own user, not with `sudo`. FTL runs as UID 1000
inside the container, so a root-owned volume causes write problems. If they were created
as root:
```bash
sudo chown -R $USER:$USER /mnt/storage/appdata
```

Secrets go in a separate file so the compose file itself can be committed:
```bash
# ~/homelab/pihole/.env
PIHOLE_PASSWORD=<strong password>
TZ=Europe/Madrid
```
```bash
chmod 600 ~/homelab/pihole/.env
```

Then bring it up:
```bash
cd ~/homelab/pihole
docker compose config      # check the file parses and .env is interpolated
docker compose up -d
docker compose ps
```

### Pi-hole v6 configuration

Version 6 renamed every setting to the `FTLCONF_*` scheme. Old v5 variables
(`WEBPASSWORD`, `DNS1`, ...) are **silently ignored** — the container starts but uses
defaults, including a randomly generated admin password. Worth checking the current
documentation rather than following older guides.

| Variable | Value used | Purpose |
|---|---|---|
| `FTLCONF_webserver_api_password` | from `.env` | Admin interface password |
| `FTLCONF_webserver_port` | `8080` | Leaves 80/443 free for a future reverse proxy |
| `FTLCONF_dns_upstreams` | `127.0.0.1#5335` | Points at Unbound. `#` separates the port, not `:` |
| `FTLCONF_dns_listeningMode` | `all` | Listen on every interface (LAN + VPN) |
| `FTLCONF_dns_dnssec` | `true` | Validate DNSSEC signatures |

The upstream value **must be quoted** in YAML: unquoted, `#` starts a comment and the
setting is lost.

### Verify

Pi-hole takes about 20 seconds on first start to build its database. Queries during that
window fail with `connection refused`, which is expected — wait for `(healthy)`:
```bash
docker compose ps
```

```bash
dig @127.0.0.1 -p 5335 google.com +short     # Unbound resolves on its own
dig @127.0.0.1 google.com +short             # full chain
dig @127.0.0.1 doubleclick.net +short        # 0.0.0.0 → blocking works
dig @10.10.0.2 google.com +short             # answers on the VPN address
```
`dig` comes from `bind9-dnsutils` (`sudo apt install dnsutils`).

The last check is the important one: it proves WireGuard clients will be able to use it.

Web interface: `http://<PI_LAN_IP>:8080/admin`

### Harmless warnings

```
WARNING: Insufficient permissions to set system time (CAP_SYS_TIME required)
```
Pi-hole v6 bundles an optional NTP client. The capability was deliberately not granted:
the host already keeps its own clock in sync.

### Integration with WireGuard

Setting `DNS = 10.10.0.2` (the Pi's VPN address) in the client profiles routes DNS
queries through the tunnel to Pi-hole. The effect: ad blocking on a phone over mobile
data, away from home, with no app installed — and no dependency on a public resolver.

### Important: do not point the Pi's own DNS at Pi-hole

Tempting, but it creates a circular dependency: if the container fails, the host loses
DNS entirely and cannot even pull images to repair it. The Pi keeps resolving through
NetworkManager and its usual upstream servers.

---

## Status

- [x] External disk formatted and mounted at `/mnt/storage` (`nofail`, by UUID).
- [x] Docker + Compose installed, image store on the external disk.
- [x] Pi-hole + Unbound, verified across a reboot.
- [ ] WireGuard client profiles pointing at Pi-hole for DNS.
- [ ] Reverse proxy (Caddy) with internal HTTPS.
- [ ] Vaultwarden (password manager).
- [ ] Backups (restic to the VPS).
- [ ] Uptime Kuma (monitoring and alerts).

## Related documents

- [SETUP.md](SETUP.md) — the VPN itself, command by command.
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — real problems and fixes, including the
  three hit while setting up this homelab (issues 11 to 13).
- [OPERATIONS.md](OPERATIONS.md) — day-to-day usage.
