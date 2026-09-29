# Pi-hole + Unbound

DNS with ad blocking, plus recursive resolution so no third party sees the browsing
history. Runs on the Raspberry Pi in Docker and is reachable **only through the VPN** —
nothing is exposed to the internet.

This document follows the same convention as [SETUP.md](SETUP.md): every command is
listed with an explanation of what it does and why.

Templates: [../pihole/docker-compose.yml.example](../pihole/docker-compose.yml.example),
[../pihole/.env.example](../pihole/.env.example) and
[../pihole/forward-records.conf.example](../pihole/forward-records.conf.example).

## Prerequisites

- Docker + Compose installed on the Pi, with the user in the `docker` group.
- Persistent data lives under `/mnt/storage/appdata/<service>` (bind-mounted), so the
  containers can be recreated without losing state. Adjust the path to wherever the
  Pi's storage is mounted.
- The WireGuard tunnel on the Pi is up, so `10.10.0.2` exists (see SETUP.md Phase 2).

---

## Why both

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

## The image does not resolve recursively by default

This is the part worth knowing before trusting the privacy claim above. Despite being
described upstream as a "validating, recursive, and caching DNS resolver", the
`mvance/unbound-rpi` image ships a `forward-records.conf` with a `forward-zone` for `"."`
pointing at Cloudflare over DNS-over-TLS, and its generated config includes it. Out of
the box the container is a **caching forwarder**, and Cloudflare sees every query.

The fix is to mount an empty file over that path, removing the forward-zone so Unbound
falls back to native recursion:

```bash
cat > /mnt/storage/appdata/unbound/forward-records.conf <<'EOF'
# Intentionally empty: removes the image's forward-zone to Cloudflare.
EOF
```
```yaml
    volumes:
      - /mnt/storage/appdata/unbound/forward-records.conf:/opt/unbound/etc/unbound/forward-records.conf:ro
```

Mount the **file**, not a directory, and over the exact path the image reads
(`/opt/unbound/etc/unbound/`). Mounting a directory somewhere else is silently inert.

No further configuration is needed: root hints are compiled into Unbound and the image's
startup script already sets `auto-trust-anchor-file` for DNSSEC.

No functional test can catch this: a forwarder and a recursive resolver both return
correct answers. Only a packet capture shows the difference (see [Verify](#verify)).

## Why the port 53 prerequisite matters

Pi-hole needs UDP/TCP port 53. Verify it is free before starting:
```bash
sudo ss -tulnp | grep :53
systemctl is-active systemd-resolved     # inactive / not-found
```
Raspberry Pi OS does not ship `systemd-resolved`, and `avahi-daemon` uses 5353 (mDNS),
a different port. On distributions that do run `systemd-resolved`, its stub listener
must be disabled first.

## Network modes: different for each container

This is the subtle part. The two containers use **different** network modes on purpose:

- **Pi-hole → `network_mode: host`.** It shares the Pi's network stack, so it listens on
  every interface at once — LAN and the WireGuard interface — and sees the real client
  IPs in its statistics. With bridge mode it would report the Docker gateway as the
  client for every query.
- **Unbound → bridge with `127.0.0.1:5335:53`.** The `mvance/unbound-rpi` image listens
  on port 53 *inside* the container. In host mode it would compete with Pi-hole for the
  host's port 53 and neither would work. The explicit mapping moves it to 5335 on the
  host, bound to loopback only so it is unreachable from the network — an open resolver
  would be abused for DNS amplification attacks.

## Deploy

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

## Pi-hole v6 configuration

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

## Verify

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

That last check is the one that proves WireGuard clients will be able to use it.

But none of the above distinguishes a recursive resolver from a forwarder — both return
correct answers. To verify **where the queries actually go**, capture the packets:

```bash
sudo tcpdump -ni any 'port 853 or port 53' -c 25
# in another terminal, query a name that cannot be cached:
dig @127.0.0.1 -p 5335 test-$(date +%s).debian.org
```

Expected output walks the hierarchy, with no port 853 traffic at all:
```
172.18.0.2 > 193.0.14.129.53   A? oRg.         <- k.root-servers.net
172.18.0.2 > 199.19.56.1.53    A? dEbiaN.org.  <- .org TLD servers
172.18.0.2 > 199.19.57.1.53    DNSKEY? org.    <- fetching keys to validate
```
The mixed case (`A? oRg.`) is DNS-0x20 anti-spoofing, applied only when Unbound resolves
for itself. Any traffic to port 853 means it is still forwarding.

And confirm DNSSEC is validated locally rather than trusted from upstream:
```bash
dig @127.0.0.1 -p 5335 dnssec-failed.org     # must return SERVFAIL
```

Expect the first lookup of a domain to take a few hundred milliseconds rather than ~0 ms.
That is the cost of walking the hierarchy instead of reading a third party's warm cache,
and it amortises as the local cache fills.

Web interface: `http://<PI_LAN_IP>:8080/admin`

## Harmless warnings

```
WARNING: Insufficient permissions to set system time (CAP_SYS_TIME required)
```
Pi-hole v6 bundles an optional NTP client. The capability was deliberately not granted:
the host already keeps its own clock in sync.

## Integration with WireGuard

Setting `DNS = 10.10.0.2` (the Pi's VPN address) in the client profiles routes DNS
queries through the tunnel to Pi-hole. The effect: ad blocking on a phone over mobile
data, away from home, with no app installed — and no dependency on a public resolver.

Applied to the **full-tunnel** profiles only. With a split tunnel, general traffic still
uses the local connection, so its own resolver is fine.

Laptop:
```bash
sudo sed -i 's/^DNS = 1.1.1.1$/DNS = 10.10.0.2/' /etc/wireguard/casa-full.conf
sudo wg-quick down casa-full; sudo wg-quick up casa-full
```

Phone: edit the DNS field of the full-tunnel entry in the WireGuard app.

Verify from the client:
```bash
cat /etc/resolv.conf           # nameserver 10.10.0.2
dig doubleclick.net +short     # 0.0.0.0  -> filtered by Pi-hole
dig google.com +short          # resolves -> answered by Unbound
curl -4 ifconfig.me            # the VPS public IP
```

The path a blocked query takes: client asks `10.10.0.2` → the query enters the WireGuard
tunnel → reaches the VPS → is routed to the Pi → Pi-hole matches it against its
blocklists. Nothing leaves the setup.

The Pi-hole admin interface should also list the clients' VPN addresses among the query
sources.

`dig` is provided by `bind` on Arch and `bind9-dnsutils` on Debian.

> If a handful of sites hang while everything else works, that is an MTU problem in the
> tunnel, not DNS. The distinction: DNS failures stop a connection from starting, MTU
> failures let it start and then stall.

## Important: do not point the Pi's own DNS at Pi-hole

Tempting, but it creates a circular dependency: if the container fails, the host loses
DNS entirely and cannot even pull images to repair it. The Pi keeps resolving through
NetworkManager and its usual upstream servers.

---

## Related documents

- [SETUP.md](SETUP.md) — the VPN itself, command by command.
- [OPERATIONS.md](OPERATIONS.md) — day-to-day usage, including Pi-hole commands.
