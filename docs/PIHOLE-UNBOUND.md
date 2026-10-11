# Raspberry Pi — Pi-hole and Unbound (DNS)

Record of the DNS service on the Raspberry Pi: Docker, Unbound as a recursive resolver,
Pi-hole as the filter in front of it, the firewall rules that open them to the LAN and the
VPN, and the switch of the FULL client profiles to Pi-hole.

## Why two services
- **Pi-hole is a filter, not a resolver.** For a blocked domain it answers `0.0.0.0`;
  every other query is passed to an **upstream** resolver. With a public upstream
  (Google, Cloudflare, Quad9) that company sees every lookup of every device.
- **Unbound is a recursive resolver on the Pi.** It asks the root servers, then the TLD
  servers (`.org`, `.de`, ...), then the domain's own authoritative servers, by itself.
  It validates DNSSEC locally. The first lookup of a domain is slower; after that the
  answer is cached.

## Design decision: recursive, not an encrypted forwarder
Two options were considered for Unbound:

- **Option A — recursive.** Unbound talks directly to root, TLD and authoritative
  servers.
- **Option B — encrypted forwarder.** Unbound forwards every query over DNS over TLS
  (DoT, port `853`) to one provider (`forward-tls-upstream: yes`).

| | A — recursive | B — DoT forwarder |
|---|---|---|
| Home ISP sees the DNS queries | yes (plain DNS, port `53`) | no |
| One company has the full DNS history | no: each server sees only its part | yes: the provider sees everything |
| Depends on a third-party resolver | no | yes |

- **Recursive and encrypted together is not practical**: authoritative servers almost
  never support encrypted DNS.
- **DoT hides less from the ISP than it seems**: the ISP still sees the destination IP
  addresses and the server name (SNI) of HTTPS connections.
- **Away from home, nothing changes**: from a café with the FULL profile, the queries
  already travel inside WireGuard to the Pi; the local network sees nothing.

Chosen: **option A**. Switching to B later only takes a few lines of Unbound
configuration.

```
clients (DNS = 10.10.0.2)
   │  port 53, through the tunnel or on the home LAN
   ▼
Pi-hole :53 (host network) ── web UI :8080
   │  127.0.0.1#5335
   ▼
Unbound 127.0.0.1:5335 (container)
   │  port 53
   ▼
root → TLD → authoritative servers
```

**Layout on the Pi**: each service has its own folder, `~/homelab/<service>/`, holding its
`docker-compose.yml`; persistent data lives under `/srv/appdata/<service>`, on the NVMe.

> **Prerequisites**:
> - [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) done: Raspberry Pi OS on NVMe, SSH hardened.
> - [RASPBERRY-WIREGUARD.md](RASPBERRY-WIREGUARD.md) done: `wg0` up and enabled at boot.
> - [RASPBERRY-FIREWALL.md](RASPBERRY-FIREWALL.md) done: table `inet pifw` loaded at boot.

Placeholders (`<PI_LAN_IP>`, `<LAN_IFACE>`, `<TIMEZONE>`, ...) are defined in the
[README](../README.md#placeholders).

---

## 1. Docker

### 1.1 Docker's apt repository
```
sudo apt update
sudo apt install ca-certificates curl
```
Refreshes the package lists and installs what is needed to download over HTTPS.

```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
Creates the keyrings folder, downloads Docker's signing key into it and makes it readable.
With this key apt verifies that the Docker packages really come from Docker.

```
. /etc/os-release && echo "$VERSION_CODENAME"
```
Prints the Debian release name the repository must match: `trixie`.

```
sudo nano /etc/apt/sources.list.d/docker.sources
```
Content (deb822 format):
```
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```
- `URIs` / `Suites` / `Components` — Docker's Debian repository, the `trixie` release, the
  `stable` channel.
- `Signed-By` — packages from this repository are accepted only if they are signed with
  that key.

### 1.2 Install
```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
Reads the new repository and installs the Docker engine, its command-line client, the
container runtime and the `docker compose` plugin (here Docker `29.x` and Compose `5.x`).

```
sudo docker run --rm hello-world
```
Downloads and runs a test container, which prints a greeting and exits; `--rm` deletes it
afterwards.

> **Security: do not add the user to the `docker` group.** Membership of that group is
> equivalent to passwordless root: any member can start a container that mounts `/`.
> Every Docker command in this document is run with `sudo`.

### 1.3 Check that nothing else changed
```
sudo nft list tables
```
Must show `inet pifw` still there, next to WireGuard's `ip filter`, `ip nat` and
`ip mangle` and the tables Docker added: `ip6 nat`, `ip6 filter` and `ip raw`.

```
sudo iptables -S FORWARD | head -3
```
Must show `-P FORWARD ACCEPT` followed by the `DOCKER-USER` and `DOCKER-FORWARD` jumps.
Docker did **not** switch the policy to `DROP`: IPv4 forwarding was already enabled by
`/etc/sysctl.d/99-vpn.conf`.

On the VPS:
```
ping -c 3 <HOME_ROUTER_IP>
```
Must still answer, with TTL `63`: the path through the Pi into the home LAN still works.

## 2. Unbound

### 2.1 Image choice
- `mvance/unbound-rpi`, the image originally planned, was last updated in January 2025
  and is built only for 32-bit ARM (`arm/v7`); `mvance/unbound` is `amd64` only.
- Chosen: **`klutchell/unbound:1.26.1`** — maintained, multi-architecture (`arm64`
  included), and small (a distroless image of about 7 MB, with no shell).
- Its default configuration is **recursive, with no forwarders**. The old trick of
  mounting an empty `forward-records.conf` over the image's file (needed with the
  `mvance` image, which forwards to Cloudflare by default) is not needed here. Recursion
  is still verified with `tcpdump` (§2.4).
- **Pin a version tag, never `latest`**: an update then happens only when the tag is
  changed on purpose.

### 2.2 Compose file
```
mkdir -p ~/homelab/unbound
nano ~/homelab/unbound/docker-compose.yml
```
Content:
```yaml
services:
  unbound:
    image: klutchell/unbound:1.26.1
    container_name: unbound
    restart: unless-stopped
    ports:
      - "127.0.0.1:5335:53/tcp"
      - "127.0.0.1:5335:53/udp"
```
- `image` — the pinned version from §2.1.
- `container_name` — a fixed name, used by `docker` commands and in this document.
- `restart: unless-stopped` — the container starts again after a crash or a reboot,
  unless it was stopped by hand.
- `ports` — publishes the container's port `53` on the Pi's port `5335`, **only on
  `127.0.0.1`**: Unbound is reachable from the Pi itself (Pi-hole), never from the LAN or
  the VPN. DNS uses UDP, and TCP for large answers, so both are published.

### 2.3 Start it
```
grep -n '<' ~/homelab/unbound/docker-compose.yml
```
Must print nothing: no placeholder is left in the file.

> **Lesson learned**: the first attempt kept a literal `<UNBOUND_TAG>` placeholder in the
> `image` line, and Compose failed with `invalid reference format`.

```
cd ~/homelab/unbound && sudo docker compose up -d
```
Downloads the image and starts the container in the background (`-d`).

```
sudo docker compose ps
```
Must show `unbound` `Up`, with ports `127.0.0.1:5335->53/tcp` and
`127.0.0.1:5335->53/udp`.

### 2.4 Test it
```
sudo apt install bind9-dnsutils tcpdump
```
Installs `dig`, to send DNS queries by hand, and `tcpdump`, to watch the packets.

```
dig @127.0.0.1 -p 5335 debian.org +short
```
Asks Unbound directly; must print IP addresses.

```
dig @127.0.0.1 -p 5335 debian.org | grep flags
```
Must show the `ad` flag in the header line: the answer was **DNSSEC-validated** by
Unbound. The second line shows the EDNS buffer size (`udp: 1232`).

```
dig @127.0.0.1 -p 5335 dnssec-failed.org | grep status
```
Must show `SERVFAIL`: this test domain has a deliberately broken signature, and Unbound
refuses to answer it. A domain without DNSSEC (for example `wikipedia.de`) answers with
no `ad` flag (observed here: flags `qr rd ra`); that is normal.

**Proof of recursion.** In a first terminal:
```
sudo tcpdump -ni <LAN_IFACE> -c 30 'port 53 or port 853'
```
Shows up to 30 DNS packets (plain on `53`, DoT on `853`) on the LAN interface (`-n`: no
name lookups).

In a second terminal:
```
dig @127.0.0.1 -p 5335 wikipedia.de
```
The sequence observed:

| Step | Query | Sent to |
|---|---|---|
| 1 | `A? de.` | a root server |
| 2 | `A? wikipedia.de` | a `.de` TLD server |
| 3 | `DNSKEY? de.` | a `.de` TLD server (for DNSSEC) |
| 4 | `A? wikipedia.de` | the domain's own authoritative servers → answer |

Everything on port `53`, to many different servers; nothing on `853` and nothing to a
single provider address. **Only `tcpdump` proves recursion**: the `dig` answer looks the
same whether Unbound recurses or forwards.

## 3. Pi-hole

### 3.1 Compose file
Version: `pihole/pihole:2026.09.0`, the latest stable release at the time, available for
`arm64`.

```
mkdir -p ~/homelab/pihole
sudo install -d -m 755 /srv/appdata/pihole
nano ~/homelab/pihole/docker-compose.yml
```
Creates the service folder and the data folder (owned by `root`), then the compose file.
Content:
```yaml
services:
  pihole:
    image: pihole/pihole:2026.09.0
    container_name: pihole
    network_mode: host
    restart: unless-stopped
    environment:
      TZ: "<TIMEZONE>"
      FTLCONF_dns_upstreams: "127.0.0.1#5335"
      FTLCONF_dns_listeningMode: "LOCAL"
      FTLCONF_webserver_port: "8080"
    volumes:
      - /srv/appdata/pihole/etc-pihole:/etc/pihole
```
- `network_mode: host` — Pi-hole uses the Pi's own network stack. It sees the clients'
  real addresses (with a bridge, every query would come from Docker's gateway) and
  listens on port `53` directly. Its ports are therefore filtered by the `pifw` `input`
  chain, unlike ports published in bridge mode
  ([RASPBERRY-FIREWALL.md §9](RASPBERRY-FIREWALL.md#9-adding-a-service-later)).
- `TZ` — the time zone, `<TIMEZONE>`, so the log timestamps are in local time.
- `FTLCONF_dns_upstreams: "127.0.0.1#5335"` — the only upstream is Unbound. Quoted,
  because of the `#`, which YAML would otherwise read as a comment. Pi-hole v6 reads only
  `FTLCONF_*` variables for its settings.
- `FTLCONF_dns_listeningMode: "LOCAL"` — Pi-hole answers only devices on networks the Pi
  is directly connected to: the home LAN and `10.10.0.0/24` through `wg0`. An extra
  barrier behind the firewall.
- `FTLCONF_webserver_port: "8080"` — the web interface listens only on `8080`, not on `80`
  or `443`.
- `volumes` — Pi-hole's configuration and lists are kept in `/srv/appdata/pihole`, so they
  survive recreating the container.

### 3.2 Start it
```
grep -n '<' ~/homelab/pihole/docker-compose.yml
```
Must print nothing: `<TIMEZONE>` must have been replaced by the real time zone.

```
cd ~/homelab/pihole && sudo docker compose up -d
sudo docker compose ps
```
Starts the container and shows its state: `(health: starting)` at first, then `(healthy)`
after about 20 seconds. **Wait for `(healthy)` before testing.**

```
sudo docker exec -it pihole pihole setpassword
```
Asks for the web interface password interactively and stores it hashed. A password in an
environment variable would be visible to anyone who can run `docker inspect`.

### 3.3 Test it on the Pi
```
dig @127.0.0.1 debian.org +short
```
Asks Pi-hole (port `53`); must print IP addresses.

```
dig @127.0.0.1 doubleclick.net +short
```
Must print `0.0.0.0`: the domain is on the default blocklist.

```
sudo docker exec pihole pihole-FTL --config dns.upstreams
```
Prints the upstream Pi-hole really uses: `[ 127.0.0.1#5335 ]`.

**Proof of the chain.** In a first terminal:
```
sudo tcpdump -ni lo -c 6 port 5335
```
Shows the traffic to Unbound's port on the loopback interface.

In a second terminal:
```
dig @127.0.0.1 wikipedia.org
```
`tcpdump` shows the query and the answer between `127.0.0.1` and `127.0.0.1.5335`:
Pi-hole asking Unbound.

### 3.4 The Pi's own resolver is not Pi-hole
```
cat /etc/resolv.conf
```
Must show the DNS servers handed out by the home router through DHCP (managed by
NetworkManager), **not** `127.0.0.1`. If the Pi used its own Pi-hole and Docker or Pi-hole
were down, the Pi would lose DNS and could not even download images to recover.

## 4. Firewall rules
Pi-hole listens on the host, so `pifw` drops its ports until they are opened
([RASPBERRY-FIREWALL.md §9](RASPBERRY-FIREWALL.md#9-adding-a-service-later)).

```
sudo nano /etc/nftables.d/pifw.nft
```
In the `input` chain, right after the SSH rule, add:
```
        udp dport 53 ip saddr @admin_nets accept
        tcp dport 53 ip saddr @admin_nets accept
        tcp dport 8080 ip saddr @admin_nets accept
```
- `udp dport 53` / `tcp dport 53` — DNS: UDP for normal queries, TCP for large answers.
- `tcp dport 8080` — the Pi-hole web interface.
- `ip saddr @admin_nets` — only from the home LAN and the VPN, the same set as SSH.

```
grep -n '<' /etc/nftables.d/pifw.nft
sudo nft -c -f /etc/nftables.d/pifw.nft && echo OK
```
No placeholder left, and the file checks without being applied
([RASPBERRY-FIREWALL.md §1.1](RASPBERRY-FIREWALL.md#11-check-before-applying)).

```
sudo systemd-run --on-active=5m --unit=pifw-revert /usr/sbin/nft delete table inet pifw
sudo nft -f /etc/nftables.d/pifw.nft
```
Arms the 5-minute revert timer, then loads the new version of the table
([RASPBERRY-FIREWALL.md §2](RASPBERRY-FIREWALL.md#2-load-it-with-a-safety-net)).

```
sudo nft list chain inet pifw input
```
Must show the three new rules after the SSH rule.

**Tests, within the 5 minutes:**

- From the laptop at home, with its VPN off: a **new** SSH session to the Pi works, and
  ```
  dig @<PI_LAN_IP> doubleclick.net +short
  ```
  prints `0.0.0.0`: Pi-hole answers on the LAN.
- In the laptop's browser, `http://<PI_LAN_IP>:8080/admin` shows the login page, and the
  password from §3.2 works.

  > On the LAN the web interface is plain HTTP: the password crosses the LAN
  > unencrypted. Prefer opening it over the VPN, `http://10.10.0.2:8080/admin`, where
  > WireGuard encrypts it.
- From the VPS (install `bind9-dnsutils` there if `dig` is missing):
  ```
  dig @10.10.0.2 debian.org +short
  ```
  prints IP addresses: Pi-hole answers through the tunnel.

Then cancel the timer:
```
sudo systemctl stop pifw-revert.timer
systemctl list-timers | grep pifw
```
The second command must print nothing.

> **Rule: test first, cancel after.** This time the timer was cancelled right after
> loading the rules, before the tests. Here that was harmless, because the change only added
> permissions; a change that removes access is exactly when the timer matters.

Nothing else is needed for persistence: this is the file `pifw.service` loads at boot.

## 5. FULL profiles → Pi-hole
The FULL profiles used a temporary public resolver (see
[CLIENTS.md](CLIENTS.md#design-points)); now they use Pi-hole, `10.10.0.2`.

**Laptop:**
```
sudo wg-quick down casa-full
```
Takes the profile down **before** editing it, so `resolvconf` removes the DNS entry that
was registered with the old value.

```
sudo nano /etc/wireguard/casa-full.conf
```
Change the `DNS` line to:
```
DNS = 10.10.0.2
```

**Phone:** in the WireGuard app, edit `casa-full` and set **DNS servers** to `10.10.0.2`.
On Android, **Private DNS** (in the network settings) must be **Off** or **Automatic**:
with a specific hostname, Android sends DNS straight to that provider and bypasses
Pi-hole.

**Test the laptop from outside home** (for example through the phone's hotspot):
```
sudo wg-quick up casa-full
```
Brings the FULL profile up.

```
resolvectl status casa-full 2>/dev/null || cat /etc/resolv.conf
```
Shows the resolver in use, as in [CLIENTS.md §8.2](CLIENTS.md#82-full). It must show
`nameserver 10.10.0.2` (or `10.10.0.2` as the interface's DNS server).

```
dig doubleclick.net +short
```
Must print `0.0.0.0`: the query went to Pi-hole through the tunnel.

```
curl -4 -s ifconfig.me ; echo
```
Must print `<VPS_PUBLIC_IP>`: internet traffic still leaves from the VPS.

```
sudo wg-quick down casa-full
```
Takes the profile down.

**Phone:** on mobile data, with `casa-full` up, browsing works.

In the Pi-hole web interface, the **Query Log** should show the clients' VPN addresses,
`10.10.0.102` (laptop) and `10.10.0.101` (phone), thanks to host networking.

## 6. Reboot test
```
sudo reboot
```
Only a real reboot proves the boot behaviour. After it:

- `systemctl is-active pifw wg-quick@wg0 docker` → `active` three times.
- `sudo nft list tables` → `inet pifw`, `ip filter`, `ip nat`, `ip mangle`, `ip6 nat`,
  `ip6 filter` and `ip raw`.
- `sudo docker ps --format '{{.Names}}  {{.Status}}'` → `pihole` `Up`
  (`health: starting`, then `healthy`) and `unbound` `Up`: both came back on their own
  (`restart: unless-stopped`).
- `dig @127.0.0.1 doubleclick.net +short` → `0.0.0.0`.
- `dig @127.0.0.1 -p 5335 debian.org | grep flags` → `ad`.
- On the VPS, `dig @10.10.0.2 debian.org +short` → IP addresses.

## 7. Updating the images
Change the pinned tag in the service's `docker-compose.yml`, then, in that folder, run
`sudo docker compose pull && sudo docker compose up -d`: it downloads the new image and
recreates the container with it. A tag can be re-published with different content; the
exact image running can be recorded by its digest with
`sudo docker image inspect --format '{{index .RepoDigests 0}}' <image>`.

---

## Current state
- Docker `29.x` from Docker's apt repository; the user is not in the `docker` group.
- Unbound `klutchell/unbound:1.26.1`, recursive with local DNSSEC validation, only on
  `127.0.0.1:5335`.
- Pi-hole `2026.09.0`, host network, web interface on `8080`, only upstream Unbound.
- `pifw` allows DNS (`53`) and the web interface (`8080`) only from the home LAN and the
  VPN.
- The FULL profiles of the phone and the laptop use `DNS = 10.10.0.2`.
- Everything survives a reboot.

Pending (optional):

- Hardening of `pifw` (an `ExecReload` in `pifw.service`, dropping IPv6 on `wg0`, invalid
  packets LAN→`wg0`).
- Reboot the VPS and confirm that the Pi reconnects on its own (keepalive).
- HTTPS for the Pi-hole web interface.

## Related documents
- [RASPBERRY-FIREWALL.md](RASPBERRY-FIREWALL.md) — the `pifw` table that the DNS and web
  rules were added to.
- [CLIENTS.md](CLIENTS.md) — the SPLIT and FULL client profiles.
- [RASPBERRY-WIREGUARD.md](RASPBERRY-WIREGUARD.md) — the Pi as WireGuard spoke, which
  carries the clients' DNS queries.
