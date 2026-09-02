# Operations manual

Day-to-day usage: bringing tunnels up and down, switching profiles, and diagnosing
problems. For the initial build see [SETUP.md](SETUP.md); for errors see
[TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Address map

| Device | VPN address | Role | Always on |
|---|---|---|---|
| VPS | `10.10.0.1` | Hub, static public IP, listens on UDP 51820 | yes |
| Raspberry Pi | `10.10.0.2` | Reverse tunnel + home LAN gateway | yes |
| Phone | `10.10.0.11` | Roaming client | on demand |
| Laptop | `10.10.0.12` | Roaming client | on demand |

VPN network: `10.10.0.0/24`. Home LAN: `<HOME_LAN_SUBNET>`.

## Profiles

| Profile | `AllowedIPs` | DNS | What it does | When to use |
|---|---|---|---|---|
| SPLIT | `10.10.0.0/24, <HOME_LAN_SUBNET>` | system default | Only home-bound traffic enters the tunnel; normal browsing keeps using the local connection | Reaching home services from outside |
| FULL | `0.0.0.0/0` | `10.10.0.2` (Pi-hole) | All traffic exits through the VPS public IP, with DNS filtered by Pi-hole | Untrusted / public Wi-Fi |

Only one profile can be active at a time on a given device.

---

## Homelab services (on the Pi)

Services are defined under `~/homelab/<service>/` and reachable only through the VPN.

```bash
cd ~/homelab/pihole
docker compose ps               # status and health
docker compose logs -f          # follow logs
docker compose restart          # restart the service
docker compose pull             # fetch newer images
docker compose up -d            # apply changes / recreate
docker compose down             # stop and remove containers
```

Across all services:
```bash
docker ps                       # everything running
docker stats                    # live CPU / RAM usage
docker images                   # images on disk
```

Wait for `(healthy)`, not just `Up`: Pi-hole needs about 20 seconds to initialize its
database on first start and refuses queries until then.

### Pi-hole

Admin interface: `http://<PI_LAN_IP>:8080/admin`

```bash
docker exec pihole pihole -g                    # update blocklists
docker exec pihole pihole -q <domain>           # why was a domain blocked
docker exec pihole pihole disable 5m            # pause blocking for 5 minutes
docker exec pihole pihole enable
```

If a site breaks, `pihole -q` shows which list matched; exceptions can be added from the
admin interface. Pausing blocking briefly is the quickest way to confirm whether Pi-hole
is the cause of a problem.

Checking the DNS chain:
```bash
dig @10.10.0.2 google.com +short            # full chain
dig @127.0.0.1 -p 5335 google.com +short    # Unbound alone (run on the Pi)
dig @10.10.0.2 doubleclick.net +short       # 0.0.0.0 -> filtering works
```

Reverting to a public resolver if Pi-hole is unavailable: change `DNS` in the client
profile back to `1.1.1.1` and restart the tunnel. The tunnel itself is unaffected.

---

## Laptop (Linux)

Bring a profile up:
```bash
sudo wg-quick up casa-split      # home LAN access
sudo wg-quick up casa-full       # everything through the VPS
```

Bring it down:
```bash
sudo wg-quick down casa-split
sudo wg-quick down casa-full
```

Switch profiles (never leave both up):
```bash
sudo wg-quick down casa-split && sudo wg-quick up casa-full
```

Check what is active:
```bash
sudo wg show
```

Optional — start a profile automatically at boot:
```bash
sudo systemctl enable wg-quick@casa-split
sudo systemctl disable wg-quick@casa-split
```

## Phone

Use the WireGuard app and toggle the tunnel you want. Activating one tunnel
deactivates the other automatically.

## Servers (VPS and Pi)

Both bring their tunnel up at boot (`systemctl enable wg-quick@wg0`), so no manual
action is normally needed.

```bash
sudo systemctl status wg-quick@wg0
sudo systemctl restart wg-quick@wg0     # reload config, recreates routes
sudo wg show
```

Restarting via `wg-quick`/systemd is the correct way to apply config changes, because
it also rebuilds routes and firewall rules from the config file.

---

## Adding a new device

1. Generate a key pair **on the new device**:
   ```bash
   sudo sh -c 'cd /etc/wireguard && umask 077 && wg genkey | tee dev_private.key | wg pubkey > dev_public.key'
   ```
2. Pick a free VPN address (e.g. `10.10.0.13`).
3. On the VPS, append the peer to `/etc/wireguard/wg0.conf`:
   ```ini
   [Peer]
   # New device
   PublicKey = <NEW_DEVICE_PUBLIC_KEY>
   AllowedIPs = 10.10.0.13/32
   ```
4. Apply it live:
   ```bash
   sudo wg set wg0 peer <NEW_DEVICE_PUBLIC_KEY> allowed-ips 10.10.0.13/32
   ```
   The file entry makes it survive reboots. If the peer routes a whole subnet (like the
   Pi does), restart with `wg-quick` instead so the route is created.
5. Configure the client with the VPS public key, endpoint and the chosen `AllowedIPs`.

## Removing a device

```bash
sudo wg set wg0 peer <DEVICE_PUBLIC_KEY> remove
```
Then delete its `[Peer]` block from `/etc/wireguard/wg0.conf` so it does not come back
after a reboot.

---

## Health checks

Is the tunnel alive:
```bash
sudo wg show
```
Look at `latest handshake`. Under a minute means healthy; several minutes with traffic
attempts means something is wrong. `transfer` should grow in both directions.

Connectivity ladder — test in this order to isolate a failure:
```bash
ping -c 3 10.10.0.1               # 1. the hub over the VPN
ping -c 3 10.10.0.2               # 2. the Pi over the VPN
ping -c 3 <HOME_ROUTER_IP>        # 3. a device on the home LAN (via the Pi)
curl -4 ifconfig.me               # 4. which public IP the world sees
```
If step 1 fails it is the tunnel itself; if only step 3 fails it is routing/NAT on the
Pi side; if step 4 shows the wrong IP the full tunnel is not active.

Confirm the full tunnel is really working:
```bash
curl -4 ifconfig.me      # must print the VPS public IP
curl -6 ifconfig.me      # must fail (IPv6 blocked, no leak)
cat /etc/resolv.conf     # must show the DNS from the profile
```

Confirm traffic reaches the LAN through the Pi:
```bash
ping -c 3 <HOME_ROUTER_IP>       # ttl=63 instead of 64 = one extra hop
```

## Useful state inspection

```bash
ip addr show wg0                        # interface and address
ip route                                # active routes
sudo iptables -S INPUT                  # firewall rules + default policy
sudo iptables -t nat -L POSTROUTING -n  # NAT / MASQUERADE
sysctl net.ipv4.ip_forward              # must be 1 on VPS and Pi
sysctl net.ipv6.conf.all.disable_ipv6   # 0 normally, 1 while full tunnel is up
systemctl is-enabled wg-quick@wg0       # starts at boot?
```

---

## Maintenance

**Server updates.** The VPS and the Pi are always on; keep them patched:
```bash
sudo apt update && sudo apt upgrade -y
```
After a kernel upgrade, reboot — otherwise new kernel modules cannot be loaded (this
caused issue 9b in the troubleshooting doc).

**Container updates.** Image versions are pinned on purpose, so updating is deliberate:
```bash
cd ~/homelab/<service>
# edit the image tag in docker-compose.yml, then:
docker compose pull
docker compose up -d
docker compose logs -f          # watch for problems
docker image prune              # reclaim space from old images
```
Persistent data lives in `/mnt/storage/appdata/<service>`, so recreating a container does
not lose state. Check the upstream release notes first: Pi-hole v5 to v6 renamed every
environment variable, and a blind update would silently fall back to defaults.

**Firewall rules.** The VPS uses a default-DROP `INPUT` policy persisted with
`netfilter-persistent`. After changing rules manually:
```bash
sudo netfilter-persistent save
```

**Verify persistence after any change.** Reboot and confirm the tunnel returns on its
own:
```bash
sudo reboot
sudo wg show
```

**Fail2ban (VPS).**
```bash
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip <IP>     # if you lock yourself out
```

## Backups

Worth keeping outside the machines:
- `/etc/wireguard/*.conf` and the key files, stored **encrypted** (a password manager
  works well). Never in git.
- The peer/address map (this file).

If the VPS is rebuilt from scratch, generating a new server key pair means every client
must be updated with the new public key. Keeping a secure backup of the VPS key avoids
reconfiguring all devices.

## Security reminders

- Private keys stay on their device; only public keys are shared.
- The repository must never contain real `.conf` files or `*.key` files — the
  `.gitignore` enforces this.
- Only two ports are reachable on the VPS: the custom SSH port and UDP 51820.
- Nothing is exposed on the home router; the Pi keeps an outbound tunnel instead.
