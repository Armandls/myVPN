# Troubleshooting

Every real problem hit while building this VPN, with the exact error message, the root
cause, the fix, and the lesson. Indexed by symptom so it can be searched quickly.

| # | Symptom | Area |
|---|---|---|
| [1](#1-ssh-keygen-no-such-file-or-directory) | `Saving key ... failed: No such file or directory` | SSH keys |
| [2](#2-ssh-copy-id-no-identities-found) | `ERROR: No identities found` | SSH keys |
| [3](#3-passwordauthentication-stays-enabled) | Password login still works after setting `no` | SSH config |
| [4](#4-error-opening-terminal-xterm-kitty) | `Error opening terminal: xterm-kitty` | Remote editors |
| [5](#5-ssh-on-the-new-port-refuses-connections) | `Connection refused` on the new SSH port | systemd socket |
| [6](#6-key-is-not-the-correct-length-or-format) | `Key is not the correct length or format` | WireGuard config |
| [7](#7-iptables-command-not-found-on-the-pi) | `iptables: command not found` | Raspberry Pi OS |
| [8](#8-the-vps-cannot-reach-the-home-lan) | 100% packet loss to the home LAN | Routing |
| [9](#9-full-tunnel-fails-on-arch-three-chained-issues) | `resolvconf: signature mismatch` / `nft: No such file or directory` | Linux client |
| [10](#10-ipv6-left-disabled-after-a-failed-tunnel-start) | IPv6 stays off after a failed `wg-quick up` | Hooks |
| [11](#11-two-dns-containers-fighting-over-port-53) | Both DNS containers unreachable, `connection refused` | Docker networking |
| [12](#12-docker-images-still-on-the-microsd-despite-data-root) | Images on the microSD although `data-root` points elsewhere | Docker storage |
| [13](#13-containers-vanish-after-a-reboot) | All containers gone after rebooting | Boot ordering |

---

## 1. ssh-keygen: No such file or directory

**Symptom**
```
Enter file in which to save the key (/home/user/.ssh/id_ed25519): ~/.ssh/ovh_vps
Saving key "~/.ssh/ovh_vps" failed: No such file or directory
```

**Root cause**
The `~` was typed **inside the interactive prompt**. Tilde expansion is done by the
shell, not by the program reading the prompt, so `ssh-keygen` treated `~` as a literal
directory name that does not exist.

**Fix**
Pass the path on the command line (where the shell expands it), or type an absolute
path at the prompt:
```bash
ssh-keygen -t ed25519 -a 100 -C "vps-key" -f ~/.ssh/<KEY_NAME>
```

**Lesson**
`~` only expands when the shell parses it. Inside prompts, use absolute paths.

---

## 2. ssh-copy-id: No identities found

**Symptom**
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed:
/usr/bin/ssh-copy-id: ERROR: No identities found
```

**Root cause**
With no key specified, `ssh-copy-id` looks for default names (`id_rsa`,
`id_ed25519`...). The key had a custom name, so nothing was found.

**Fix**
```bash
ssh-copy-id -i ~/.ssh/<KEY_NAME>.pub <USER>@<VPS_PUBLIC_IP>
```

**Lesson**
Custom key names must be passed explicitly with `-i`, both to `ssh-copy-id` and to
`ssh` (or configured via `IdentityFile` in `~/.ssh/config`).

---

## 3. PasswordAuthentication stays enabled

**Symptom**
`/etc/ssh/sshd_config.d/` contained two files with contradictory values:
```
50-cloud-init.conf        → PasswordAuthentication yes
60-cloudimg-settings.conf → PasswordAuthentication no
```
and the effective value was `yes`.

**Root cause**
Drop-in files are read in lexicographic order, and for most directives **sshd keeps
the first value it sees**. `50-` is read before `60-`, so the `yes` won. Adding a
`99-hardening.conf` with `no` would not have helped either — it loads even later.

**Fix**
Remove both provider files, add an explicit hardening drop-in, and stop cloud-init
from regenerating one:
```bash
# /etc/cloud/cloud.cfg.d/99-disable-ssh-config.cfg
ssh_pwauth: false
```
```bash
# /etc/ssh/sshd_config.d/99-hardening.conf
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
```
Always verify the **effective** configuration rather than reading files:
```bash
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
```

**Lesson**
In sshd, first match wins — the opposite of many other config systems. Verify with
`sshd -T`, and remember cloud images may recreate their own drop-ins.

---

## 4. Error opening terminal: xterm-kitty

**Symptom**
```
sudo nano /etc/wireguard/wg0.conf
Error opening terminal: xterm-kitty.
```
Also seen as `ncurses: cannot initialize terminal type ($TERM="xterm-kitty")`.

**Root cause**
The local terminal (kitty) sets `TERM=xterm-kitty`, but the remote host has no
terminfo entry for it, so ncurses-based programs cannot initialize.

**Fix — quick, one-off**
```bash
sudo TERM=xterm-256color nano /path/to/file
```

**Fix — permanent, system-wide**
```bash
# from the local machine
infocmp -x xterm-kitty | ssh <HOST> 'sudo tic -x -'
```
`tic` prints a harmless warning about the description field; it still installs.

Note that installing it only for your user is not enough: `sudo nano` runs as root and
needs the entry in the system terminfo database (`/usr/share/terminfo/`).

**Lesson**
Terminal emulators with non-standard `TERM` values need their terminfo shipped to every
remote host, installed system-wide so root sees it too.

---

## 5. SSH on the new port refuses connections

**Symptom**
After changing the SSH port via a `ssh.socket` override, the server was listening but
clients got:
```
ssh: connect to host <VPS_PUBLIC_IP> port <SSH_PORT>: Connection refused
```
`ss` showed only:
```
LISTEN 0 4096 [::]:<SSH_PORT> [::]:*
```

**Root cause**
The override contained a single `ListenStream=<SSH_PORT>`, which systemd bound to
IPv6 only. Connections arriving over IPv4 had nothing listening, hence *refused*
(not a timeout, which would suggest a firewall).

**Fix**
Declare both families explicitly:
```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:<SSH_PORT>
ListenStream=[::]:<SSH_PORT>
```
The empty `ListenStream=` resets the inherited port 22. Then:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo ss -tlnp | grep <SSH_PORT>     # two lines expected
```

**Lesson**
`Connection refused` means something answered "nobody is listening" — a socket/bind
problem. A firewall drop would instead produce a timeout. On recent Ubuntu the SSH port
lives in `ssh.socket`, not `sshd_config`.

---

## 6. Key is not the correct length or format

**Symptom A** — leftover editor keystrokes
```
wg setconf wg0 /dev/fd/63
Key is not the correct length or format: `<44-char-key>=wq'
Configuration parsing error
```

**Root cause A**
The `wq` at the end is a vim save-and-quit command that was typed **into the buffer**
instead of being executed, corrupting the key. A valid WireGuard key is 44 base64
characters ending in `=`, with nothing after it.

**Symptom B** — unexpanded shell variable
```
Key is not the correct length or format: `$PRIV'
```

**Root cause B**
The config was generated with a heredoc using `\$PRIV`. Inside an **unquoted** heredoc
(`<<EOF`) the shell already expands variables, so escaping it wrote the literal text
`$PRIV` into the file.

**Fix**
Use `$PRIV` (no backslash) and verify the file afterwards:
```bash
sudo grep PrivateKey /etc/wireguard/<name>.conf | cut -d' ' -f3 | wg pubkey
```
This derives the public key from whatever is in the file; it must match the public key
registered on the peer. It also validates the format in one step.

**Lesson**
Always verify a key by deriving its public counterpart rather than eyeballing it.
`<<EOF` expands variables; `<<'EOF'` does not.

---

## 7. iptables: command not found on the Pi

**Symptom**
```
[#] iptables -t nat -A POSTROUTING -o <LAN_IFACE> -j MASQUERADE
/usr/bin/wg-quick: line 295: iptables: command not found
[#] ip link delete dev wg0
```
`wg-quick` treats a failing `PostUp` as fatal and tears the interface down.

**Root cause**
Recent Raspberry Pi OS uses nftables and does not ship the `iptables` binary, which
`wg-quick` invokes for the NAT rule.

**Fix**
```bash
sudo apt install iptables
```
This provides `iptables-nft` (iptables syntax translated to nftables), fully
compatible with the rules used here and consistent with the VPS configuration.

**Lesson**
A failing hook aborts the whole tunnel. Distros are migrating to nftables, so do not
assume `iptables` exists.

---

## 8. The VPS cannot reach the home LAN

**Symptom**
The tunnel was healthy (handshake in both directions, `ping 10.10.0.2` fine) but:
```
ping -c 3 <HOME_ROUTER_IP>
3 packets transmitted, 0 received, 100% packet loss
```
Meanwhile on the VPS:
```
ip route | grep <HOME_LAN_SUBNET>
(empty)
```

**Root cause**
The Pi peer had been added with `wg set`. That registers the peer and its
`allowed-ips` inside WireGuard, but **does not create the kernel route** to the LAN
subnet. Without a route the VPS had no idea it should send that traffic to `wg0`.

**Fix — quick test**
```bash
sudo ip route add <HOME_LAN_SUBNET> dev wg0
```

**Fix — persistent (preferred)**
Keep the `[Peer]` block in `wg0.conf` and let `wg-quick` build everything:
```bash
sudo systemctl restart wg-quick@wg0
ip route | grep <HOME_LAN_SUBNET>     # now shows 'dev wg0'
```
`wg-quick` derives routes from each peer's `AllowedIPs`, and they are recreated on
every boot.

**Lesson**
`wg set` configures WireGuard; `wg-quick` configures WireGuard **and** the routing
table. For anything involving routed subnets, use `wg-quick`.

Diagnostic tip: a reply with `ttl=63` instead of `64` proves the packet crossed one
extra hop, confirming it was routed through the Pi.

---

## 9. Full tunnel fails on Arch: three chained issues

Bringing up `AllowedIPs = 0.0.0.0/0` on the laptop failed three times for three
different reasons.

### 9a. resolvconf missing

**Symptom**
```
[#] resolvconf -a casa-full -m 0 -x
/usr/bin/wg-quick: line 32: resolvconf: command not found
[#] ip link delete dev casa-full
```

**Root cause**
The `DNS =` directive makes `wg-quick` call `resolvconf`, which was not installed.

**Fix**
```bash
sudo pacman -S openresolv
```

### 9b. Kernel modules out of sync

**Symptom**
```
[#] nft -f /dev/fd/63
/dev/fd/63:5:82-95: Error: Could not process rule: No such file or directory
```
Repeated for the rules using `fib` and `ct mark`.

**Root cause**
A full tunnel makes `wg-quick` add extra nftables rules (kill-switch) that need the
`nft_fib_ipv4` and `nft_ct` kernel modules. Diagnosis:
```bash
uname -r                  # 7.1.8-arch1-3   ← running kernel
ls /usr/lib/modules/      # 7.1.9-arch1-2   ← only the new modules on disk
```
The kernel had been upgraded without rebooting. Already-loaded modules kept working,
but no **new** module could be loaded, so nftables could not instantiate those rules.

**Fix**
```bash
sudo reboot
```

**Lesson**
On rolling-release distros, a kernel upgrade without a reboot silently breaks loading
of any module not already in memory. `No such file or directory` from `nft` usually
means a missing kernel module, not a missing file.

### 9c. NetworkManager overwriting resolv.conf

**Symptom**
```
[#] resolvconf -a casa-full -m 0 -x
resolvconf: signature mismatch: /etc/resolv.conf
resolvconf: run `resolvconf -u` to update
```
And `/etc/resolv.conf` starting with `# Generated by NetworkManager`.

**Root cause**
NetworkManager was writing `/etc/resolv.conf` itself, so openresolv found a file
without its signature and refused to touch it. Setting only `dns=resolvconf` was not
enough: it selects the DNS backend but leaves NetworkManager as the file's owner
(`rc-manager=symlink` by default). The setting appeared to work until the next reboot,
when NetworkManager recreated the file.

**Fix**
```ini
# /etc/NetworkManager/conf.d/rc-manager.conf
[main]
dns=resolvconf
rc-manager=resolvconf
```
```bash
sudo systemctl restart NetworkManager
sudo resolvconf -u
cat /etc/resolv.conf      # must read "Generated by resolvconf"
```
Verify it **survives a reboot** — that is where the incomplete version failed.

Rollback if DNS breaks:
```bash
sudo rm /etc/NetworkManager/conf.d/rc-manager.conf
sudo systemctl restart NetworkManager
```

**Lesson**
`dns=` and `rc-manager=` are different knobs; both are required. Any change to DNS
management must be validated across a reboot, not just right after applying it.

---

## 10. IPv6 left disabled after a failed tunnel start

**Symptom**
After a failed `wg-quick up`, IPv6 stayed off system-wide:
```bash
sysctl net.ipv6.conf.all.disable_ipv6
net.ipv6.conf.all.disable_ipv6 = 1
```

**Root cause**
IPv6 was being disabled from `PreUp` to prevent leaks in full-tunnel mode. When
`wg-quick` failed later (DNS, nftables...), it aborted before ever running `PostDown`,
so the restore step never executed.

**Fix**
Move the blocking hooks from `PreUp` to `PostUp`, so IPv6 is only touched once
everything else has succeeded:
```ini
PostUp   = sysctl -w net.ipv6.conf.all.disable_ipv6=1
PostUp   = sysctl -w net.ipv6.conf.default.disable_ipv6=1
PostDown = sysctl -w net.ipv6.conf.all.disable_ipv6=0
PostDown = sysctl -w net.ipv6.conf.default.disable_ipv6=0
```
Manual recovery if it happens:
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0
```

**Lesson**
Hooks that change global system state should run as late as possible, so a partial
failure cannot leave the system in a modified state. `PreUp` changes stick around if
the tunnel never comes up.

---

## 11. Two DNS containers fighting over port 53

**Symptom**
Both Pi-hole and Unbound reported as `Up`, yet neither answered:
```
dig @127.0.0.1 -p 5335 google.com
;; communications error to 127.0.0.1#5335: connection refused

dig @127.0.0.1 google.com
;; communications error to 127.0.0.1#53: timed out
```

**Root cause**
The `mvance/unbound-rpi` image listens on **port 53 inside the container**; it is
designed to be run in bridge mode with a `5335:53` mapping. Putting it in
`network_mode: host` made it try to bind the *host's* port 53 — the same port Pi-hole
needs. Both fought for it and neither ended up serving correctly.

Nothing was listening on 5335 at all, which is what `connection refused` indicated.

**Fix**
Keep Pi-hole in host mode (it must see every interface and the real client IPs), but run
Unbound in bridge mode with an explicit mapping:
```yaml
  unbound:
    image: mvance/unbound-rpi:1.22.0
    ports:
      - "127.0.0.1:5335:53/udp"
      - "127.0.0.1:5335:53/tcp"
```
Binding to `127.0.0.1` also keeps the recursive resolver unreachable from the network,
which matters because an open resolver is abused for DNS amplification attacks.

```bash
docker compose down && docker compose up -d
dig @127.0.0.1 -p 5335 google.com +short
```

**Lesson**
`network_mode: host` is not a drop-in replacement for port mappings. Check which port an
image listens on internally before switching it to host mode, especially when two
containers provide the same kind of service.

---

## 12. Docker images still on the microSD despite data-root

**Symptom**
`data-root` was set to the external disk and `docker info` confirmed it, but the disk was
nearly empty while images clearly existed:
```
docker info | grep "Docker Root Dir"   → /mnt/storage/docker
sudo du -sh /mnt/storage/docker          → 656K
docker images                            → ~445 MB of images
sudo du -sh /var/lib/containerd          → 490M      ← on the microSD
```

**Root cause**
Docker 29.x enables the **containerd snapshotter** by default:
```
msg="Starting daemon with containerd snapshotter integration enabled"
storage-driver=overlayfs
```
With it, images are stored by **containerd** under `/var/lib/containerd`, outside
Docker's `data-root`. Setting `data-root` therefore moves volumes and build cache but
**not the images** — the bulk of the data, and the whole point of using an external disk
to spare the flash card.

**Fix**
Move containerd's store as well:
```bash
sudo systemctl stop docker.socket docker.service containerd
sudo mkdir -p /mnt/storage/containerd
sudo rsync -aHAX --info=progress2 /var/lib/containerd/ /mnt/storage/containerd/
sudo du -sh /mnt/storage/containerd     # verify the size matches the original
```
```bash
sudo tee /etc/containerd/config.toml > /dev/null <<'EOF'
version = 2
root = "/mnt/storage/containerd"
state = "/run/containerd"
EOF
sudo systemctl start containerd docker
docker images                            # images must still be listed
```
Only then reclaim the space:
```bash
sudo mv /var/lib/containerd /var/lib/containerd.old
```

`-H -A -X` on rsync are not optional: containerd depends on hard links and extended
attributes for image layers.

Keep `state` under `/run` (tmpfs). It is meant to be volatile and writing it to disk
would add wear for no benefit.

**A mistake worth avoiding**: the rsync destination was mistyped the first time
(`/mnt/storage/containe`), so 490 MB landed in a stray directory while containerd
started with an empty store and every image disappeared from `docker images`. The
give-away was `du` on the intended path reporting a few hundred KB while `df` showed
500 MB used on the disk. Always confirm with `du -sh` on the destination before starting
the services.

**Lesson**
On Docker 29+, `data-root` no longer governs image storage. Verify where the space is
actually consumed with `du`, rather than trusting a configuration setting.

---

## 13. Containers vanish after a reboot

**Symptom**
After rebooting the Pi, containers were not just stopped — they were **gone**:
```
docker compose ps -a
NAME  IMAGE  COMMAND  SERVICE  CREATED  STATUS  PORTS
(empty)
```
Docker itself was `active`, the disk was mounted, and DNS queries failed with
`connection refused`.

**Root cause**
A boot race. Docker and containerd store their data on an **external USB disk**, which
takes a moment to be detected and mounted. Both daemons started first, found nothing at
their configured paths, and initialised an empty store. The `nofail` mount option — which
correctly prevents a missing disk from blocking the boot — means the system carries on
regardless.

**Fix**
Declare the dependency so systemd starts them only after the mount:
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
`RequiresMountsFor=` makes systemd resolve which mount unit covers that path and treat it
as a dependency automatically.

Verify with a real reboot:
```bash
sudo reboot
# after reconnecting, with no manual intervention:
docker compose ps                        # must be Up (healthy)
sudo du -sh /mnt/storage/containerd      # data still on the disk
```

**Note on `systemctl edit`**: it may refuse to save with
`after editing, new contents are empty, not writing file`, which happens when the editor
exits without writing anything. Writing the override file directly with `tee` avoids the
problem entirely.

**Lesson**
Any service whose data lives on removable or slow-to-appear storage needs an explicit
`RequiresMountsFor`. And a change to boot behaviour is only proven by actually rebooting.

---

## Note on start-up timing

Several of these looked like failures but were just impatience. Pi-hole spends roughly
20 seconds building its database on first start, and answers `connection refused` until
it is ready. Check the health state before concluding something is broken:
```bash
docker compose ps        # wait for (healthy), not just Up
docker compose logs -f
```

---

## General debugging checklist

VPN:
```bash
sudo wg show                          # handshake, transfer, allowed ips
ip addr show <iface>                  # is the interface up with the right address
ip route                              # are the expected routes present
sudo iptables -S INPUT                # firewall rules and default policy
sudo iptables -t nat -L POSTROUTING -n
sysctl net.ipv4.ip_forward            # forwarding enabled
cat /etc/resolv.conf                  # who owns DNS right now
curl -4 ifconfig.me                   # which IP the world sees
```

Containers and DNS services:
```bash
docker compose ps                     # Up AND healthy?
docker compose logs --tail 50
sudo ss -tulnp | grep ':53 '          # who actually holds port 53
dig @127.0.0.1 google.com +short      # does the chain resolve
dig @127.0.0.1 doubleclick.net +short # 0.0.0.0 means blocking is active
docker info | grep "Docker Root Dir"
sudo du -sh /mnt/storage/* /var/lib/containerd   # where the space really is
```

Reading the signals:
- **No handshake** → connectivity or key mismatch: check `Endpoint`, UDP 51820 open,
  and that each side has the other's correct public key.
- **Handshake but no traffic** → routing or NAT: check `AllowedIPs`, `ip route`,
  `MASQUERADE`, and `ip_forward`.
- **`Connection refused`** → nothing listening (socket/bind), or a service still starting.
- **Timeout / 100% loss** → dropped by firewall or no route.
- **`ttl` one lower than expected** → traffic crossed an extra hop (working as
  intended when routing through a gateway).
- **Container `Up` but not answering** → check `healthy` state and its logs; it may still
  be initialising.
- **Config set but no effect** → verify the actual state (`du`, `ss`, `info`) instead of
  trusting the configuration file.
