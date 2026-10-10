# VPS — WireGuard hub

Record of the WireGuard setup on the VPS: the hub of the VPN, documented here without
peers (they are added in their own documents).

The VPN is hub-and-spoke on `10.10.0.0/24`. The VPS (`10.10.0.1`) is the hub: it is the
only peer with a public IP and it listens on `51820/udp`. Every other peer (Raspberry Pi
`10.10.0.2`, phone `10.10.0.11`, laptop `10.10.0.12`) connects to it. The peers are added
later, each in its own document.

> **Prerequisite**: [VPS-SETUP.md](VPS-SETUP.md) done: SSH hardening and the base
> firewall ([§7](VPS-SETUP.md#7-firewall-iptables-ipv4-and-ipv6)), with `INPUT` and
> `FORWARD` on `DROP` for IPv4 and IPv6, and `51820/udp` already allowed in `INPUT` for
> both.

## Design decision: IPv4-only tunnel
The tunnel carries **only IPv4**. There are no IPv6 addresses inside the tunnel, and the
IPv6 `FORWARD` chain on the VPS stays at `DROP`.

When a client sends all its traffic through the VPN (FULL profile), there are three ways
to handle IPv6:

- **Leak**: the tunnel carries only IPv4 and the client's IPv6 is left alone. IPv6
  traffic goes out directly through the local network, outside the tunnel, with the
  client's real address. This defeats the purpose of the FULL profile.
- **Blocked**: the tunnel carries only IPv4 and the client blocks IPv6. Browsers and apps
  fall back to IPv4 within a moment, which goes through the tunnel.
- **Full dual stack**: the tunnel also carries IPv6. It needs private IPv6 (ULA)
  addresses for the peers, NAT66 on the VPS, `ip6tables` hooks mirroring the IPv4 ones,
  and care with `accept_ra` (enabling IPv6 forwarding changes how the kernel handles
  Router Advertisements, which the VPS may need for its own IPv6 address).

IPv4-only with IPv6 blocked on the clients was chosen: it is much simpler, and nearly
every site is dual-stack, so IPv4 alone reaches it. The consequence is that the clients'
FULL profiles **must capture or block IPv6** so it does not leak outside the tunnel. That
is handled in the client documents.

---

## 1. Install WireGuard
```
sudo apt install wireguard
```
Installs the `wireguard` package, which pulls in `wireguard-tools` (the `wg` and
`wg-quick` commands). The WireGuard module itself is already part of the kernel, so
nothing else needs to be built.

```
wg --version
```
Prints the version of `wireguard-tools`, confirming the install.

## 2. Persistent IPv4 forwarding
The hub must route packets between peers and out to the internet, so the kernel must
have IPv4 forwarding enabled.

```
cat /proc/sys/net/ipv4/ip_forward
```
Shows the current value in the running kernel (`1` = enabled). On this VPS it was already
`1`, even though no file written by the admin sets it: the cloud image enables it on its
own (see [VPS-SETUP.md §7.4](VPS-SETUP.md#74-default-policies)). Because the VPN depends
on it, it is made an explicit decision in a file of our own, instead of relying on an
image default that could change.

Where **not** to set it:

- Ubuntu 26.04 has no `/etc/sysctl.conf`.
- `/etc/ufw/sysctl.conf` exists, but it is read only by ufw, which is not installed.
  Editing it has no effect; do not use it. If it was edited, revert the change.
- `/etc/sysctl.d/99-cloudimg-ipv6.conf` is written by the cloud-image build. It disables
  IPv6 temporary (privacy) addresses, so the server keeps a stable IPv6 address. Leave it
  as is.

Create `/etc/sysctl.d/99-vpn.conf`:
```
sudo nano /etc/sysctl.d/99-vpn.conf
```
Content:
```
# file: /etc/sysctl.d/99-vpn.conf
# The VPS is the WireGuard hub: route packets between peers and to the internet
net.ipv4.ip_forward = 1
```
Files in `/etc/sysctl.d/` are applied in lexical order. `99-vpn.conf` comes after
`99-cloudimg-ipv6.conf`, so if both ever set the same key, this file wins.

> `sudo sysctl -w net.ipv4.ip_forward=1` changes only the running kernel and is lost at
> reboot. That is why the setting goes in a file.

Apply and verify:
```
sudo sysctl --system
```
Re-applies every sysctl file, printing each file it reads and each value it sets. The
output must include `* Applying /etc/sysctl.d/99-vpn.conf ...` followed by
`net.ipv4.ip_forward = 1`. That line is the proof that the file is read: checking only
the value is not, because the image already sets it to `1`.

Then reboot (`sudo reboot`) and check `cat /proc/sys/net/ipv4/ip_forward` again: it must
still be `1`.

> `sysctl -a` without `sudo` prints harmless `permission denied` messages for keys only
> `root` can read. Use `sudo` or ignore them.

IPv6 forwarding is left as the image sets it (enabled). It is harmless: the `ip6tables`
`FORWARD` policy is `DROP`, so nothing is routed over IPv6. Optionally, it can be turned
off explicitly by adding `net.ipv6.conf.all.forwarding = 0` to the same file.

## 3. Key pair
```
sudo sh -c 'cd /etc/wireguard && umask 077 && wg genkey | tee privatekey | wg pubkey > publickey'
```
Generates the VPS key pair as `root`, inside `/etc/wireguard`.

- `sudo sh -c '...'` — runs the whole line in a root shell. With plain `sudo wg genkey >
  file`, the redirection would be done by the normal user's shell, not by root.
- `umask 077` — files created from now on in this shell get permissions `600` (read and
  write for the owner only). Without it, root creates them as `644`, and the private key's
  secrecy would rest only on the permissions of the `/etc/wireguard` directory.
- `wg genkey | tee privatekey` — generates a private key, saves it in `privatekey` and
  passes it on.
- `wg pubkey > publickey` — derives the public key from the private key and saves it.

```
sudo ls -l /etc/wireguard/
```
Every file must show `-rw------- ... root root`.

The **public key** is what the peers put in their configuration to recognise the VPS.
The **private key** never leaves the VPS.

## 4. Identify the public interface
```
ip route show default
```
Shows the default route, the one used to reach the internet. The word after `dev` is the
name of the public interface: it is `<PUBLIC_IFACE>` in the configuration below.

## 5. Configuration: `/etc/wireguard/wg0.conf`
```
sudo nano /etc/wireguard/wg0.conf
```
Opens the tunnel configuration as `root`.

Content:
```
[Interface]
Address    = 10.10.0.1/24
PrivateKey = <VPS_PRIVATE_KEY>
ListenPort = 51820

PostUp   = iptables -A FORWARD -i %i -j ACCEPT
PostUp   = iptables -A FORWARD -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostUp   = iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o <PUBLIC_IFACE> -j MASQUERADE
PostUp   = iptables -t mangle -A FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o <PUBLIC_IFACE> -j MASQUERADE
PostDown = iptables -t mangle -D FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```
`<VPS_PRIVATE_KEY>` is the content of `/etc/wireguard/privatekey`.

```
sudo chmod 600 /etc/wireguard/wg0.conf
```
Read and write for `root` only: the file contains the private key. `nano` creates new
files as `644`.

### 5.1 The `[Interface]` section
- `Address = 10.10.0.1/24` — the VPS address inside the tunnel, and the size of the VPN
  network. `10.10.0.0/24` is used instead of the more obvious `10.0.0.0/24` because that
  one is common on home and café networks: if the local network and the VPN used the same
  range, routing would be ambiguous.
- `PrivateKey` — the VPS private key.
- `ListenPort = 51820` — the UDP port the hub listens on, already allowed in `INPUT` by
  the base firewall.
- No `DNS` line: that setting is for clients, to choose their resolver while connected.
  The server keeps its own resolver.
- No `SaveConfig`: with `SaveConfig = true`, `wg-quick` rewrites the file on shutdown
  with the live state, which would overwrite manual edits and comments.

### 5.2 How the hooks work
- `PostUp` lines are run by `wg-quick` with bash, as `root`, **after** the interface is
  up; `PostDown` lines **after** it is taken down. Each line is one command, run in order.
- `%i` is replaced by the interface name, `wg0`.
- `-A` appends a rule to a chain; `-D` deletes the rule with exactly the same
  specification. Every `PostUp -A` therefore has a mirrored `PostDown -D`, so taking the
  tunnel down leaves the firewall exactly as it was.
- `iptables` has several tables: `filter` (the default when no `-t` is given: accept or
  drop), `nat` (address rewriting) and `mangle` (modifying packet headers).
- `FORWARD` handles packets that **pass through** the VPS (from one interface to
  another), while `INPUT` handles packets **addressed to** the VPS itself. VPN traffic
  going to the internet or to another peer is `FORWARD` traffic.
- `POSTROUTING` (in `nat`) is the last step before a packet leaves an interface, after
  the routing decision: the place where the source address is rewritten.

### 5.3 The four rules
**Rule 1 — `FORWARD -i %i -j ACCEPT`.** Lets traffic **coming from the tunnel** be
routed: to the internet (FULL profile) and between peers (`wg0` → `wg0`, for example
laptop → Pi → home LAN). It is safe to accept everything from `wg0`, because WireGuard
only accepts packets encrypted with a known peer key and whose source address is inside
that peer's `AllowedIPs`. Every peer can therefore reach every other peer and, once the
Pi is added, the home LAN; this is intended in this design.

**Rule 2 — `FORWARD -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`.** Lets
traffic **going into the tunnel** through only if it belongs to a connection a peer
already started: replies come back, but the internet can never start a connection
towards a peer. Anything else heading into `wg0` falls through to the `FORWARD DROP`
policy.

**Rule 3 — `-t nat POSTROUTING -s 10.10.0.0/24 -o <PUBLIC_IFACE> -j MASQUERADE`.**
Packets from peers carry a private source address (`10.10.0.x`) that the internet cannot
reply to. `MASQUERADE` rewrites the source to the VPS public IP when the packet leaves,
so replies come back to the VPS; connection tracking then reverses the rewrite and
delivers the reply to the right peer.

- `-s 10.10.0.0/24` — only VPN traffic is NATed, never the VPS's own traffic.
- `-o <PUBLIC_IFACE>` — only traffic leaving to the internet is NATed. Traffic between
  peers leaves through `wg0` and is not touched, so the Pi sees the real address of the
  client (`10.10.0.12`, not the VPS).
- `MASQUERADE` vs `SNAT`: `SNAT --to-source <address>` needs a fixed address written in
  the rule; `MASQUERADE` uses whatever address the interface has at that moment, so the
  rule needs no IP.

**Rule 4 — `-t mangle FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS
--clamp-mss-to-pmtu`.** WireGuard adds its own headers to each packet, so `wg0` has an MTU
of `1420` instead of the usual `1500`. When a TCP connection opens, each side announces
its MSS: the largest segment it is willing to **receive**. A peer already announces
`1420 − 40` (IPv4 + TCP headers) = `1380`, because its own `wg0` has MTU `1420`, and it
also caps what it sends to the MTU of its route. So for traffic between a peer and the
internet this rule is a safety net. It matters for hosts that do not know about the
tunnel, such as devices on the home LAN behind the Pi (MTU `1500`, MSS `1460`), when Path
MTU discovery is broken because ICMP "fragmentation needed" is blocked. The typical
symptom is "small pages load, big ones hang".

The rule rewrites the MSS in TCP packets that open a connection (`SYN` set, `RST` not
set: the `SYN` and `SYN-ACK`) leaving through `wg0`, lowering it to the route MTU minus
40 (`1380`). The host that receives that packet then never sends segments larger than the
tunnel can carry. With `-o %i` this covers replies going into the tunnel and all
peer-to-peer traffic (`wg0` → `wg0`), which also leaves through `wg0`.

### 5.4 Packet journey
A laptop in a café, on the FULL profile, opens a website:
```
laptop (10.10.0.12)
   │  encrypted, over UDP to <VPS_PUBLIC_IP>:51820
   ▼
VPS wg0 ── FORWARD -i wg0 ACCEPT ............................ rule 1
   │
   ▼
POSTROUTING: source 10.10.0.12 → <VPS_PUBLIC_IP> (MASQUERADE) rule 3
   │  out through <PUBLIC_IFACE>
   ▼
website
   │  reply to <VPS_PUBLIC_IP>
   ▼
VPS: conntrack reverses the NAT: destination → 10.10.0.12
   │
   ▼
mangle FORWARD -o wg0: MSS clamped in the SYN-ACK ........... rule 4
   │
   ▼
filter FORWARD -o wg0 RELATED,ESTABLISHED ACCEPT ............ rule 2
   │  encrypted, back through wg0
   ▼
laptop
```

### 5.5 What is deliberately not there
- **No `ip6tables` hooks**: the tunnel is IPv4-only (see the design decision above), and
  the IPv6 `FORWARD` chain stays at `DROP`.
- **No `INPUT` hook**: the base firewall already allows `51820/udp`. `ping 10.10.0.1` and
  SSH to the VPS over the tunnel are traffic addressed to the VPS itself, and they pass the
  existing `INPUT` rules (ICMP echo request, `<SSH_PORT>`/tcp).

> Replace the placeholders with the real values in the file on the VPS. A literal
> `<PUBLIC_IFACE>` makes bash read `<` as input redirection: the `PostUp` fails with
> `No such file or directory`.

## 6. Pre-flight checks
Before bringing the tunnel up for the first time.

```
sudo iptables -S
sudo iptables -t nat -S
sudo iptables -t mangle -S
```
Lists the rules of each table. `filter` must show `INPUT DROP` with the base allow rules
from [VPS-SETUP.md §7.3](VPS-SETUP.md#73-allow-rules), and `FORWARD` only
`-P FORWARD DROP`. `nat` and `mangle` must show only their policies. Leftover rules would
otherwise end up duplicated next to the ones added by `PostUp`.

```
sudo grep PrivateKey /etc/wireguard/wg0.conf | cut -d' ' -f3 | wg pubkey
sudo cat /etc/wireguard/publickey
```
The first command extracts the private key from the config and derives its public key;
the second prints the saved public key. Both outputs must be identical: that proves the
key pasted in `wg0.conf` is the right one. `cut -d' ' -f3` takes the third
space-separated field, so it assumes the line is written exactly as
`PrivateKey = <key>`, with single spaces.

```
sudo ls -l /etc/wireguard/
```
Every file, `wg0.conf` included, must be `-rw------- root root`.

Manual symmetry test:
```
sudo wg-quick up wg0
```
Brings the tunnel up by hand and runs the `PostUp` lines.

```
sudo iptables -S FORWARD
sudo iptables -t nat -S POSTROUTING
sudo iptables -t mangle -S FORWARD
```
Together they must show the four hook rules: two in `filter FORWARD`, one in
`nat POSTROUTING`, one in `mangle FORWARD`.

```
sudo wg-quick down wg0
```
Takes the tunnel down and runs the `PostDown` lines. It must print no `Bad rule` errors,
and the same three commands must again show only the policies. This proves that every
`PostUp` has a matching `PostDown`.

## 7. Start at boot with systemd
A tunnel brought up with `wg-quick up` does not survive a reboot, and the hub must come
up unattended. `wireguard-tools` ships the template unit `wg-quick@.service`, so there is
nothing to create: the part after `@` is the interface name, and `wg-quick@wg0` uses
`/etc/wireguard/wg0.conf`.

```
sudo systemctl enable --now wg-quick@wg0
```
`enable` starts the tunnel at every boot; `--now` also starts it right away.

From now on, manage the tunnel with `sudo systemctl stop|start|restart wg-quick@wg0`, not
with `wg-quick` directly. Mixing the two leaves systemd out of sync: it reports the
service `active` while the tunnel is down, or fails to start it with
`wg0 already exists`.

## 8. Verify
```
systemctl is-enabled wg-quick@wg0
```
Must print `enabled`: the tunnel starts at boot.

```
systemctl status wg-quick@wg0 --no-pager
```
Must show `active (exited)`. This is normal: `wg-quick` configures the interface and
exits, and the kernel keeps the tunnel running.

```
sudo wg show
```
Shows `interface: wg0`, its public key, `listening port: 51820` and no peers.

```
ip -4 addr show wg0
```
Must show `inet 10.10.0.1/24` and `mtu 1420`.

```
sudo iptables -S FORWARD
sudo iptables -t nat -S POSTROUTING
sudo iptables -t mangle -S FORWARD
```
Each of the four hook rules must appear **exactly once**. A duplicate means a hook ran
twice without a clean `down` in between (see §10).

```
sudo ss -ulpn | grep 51820
```
Must show UDP sockets on `0.0.0.0:51820` and `[::]:51820`, with no process name: the
socket belongs to the kernel, not to a program.

```
sudo ip6tables -S FORWARD
```
Must print only `-P FORWARD DROP`: nothing is routed over IPv6.

## 9. Reboot test
```
sudo reboot
```
Only a real reboot proves the boot behaviour. After reconnecting, repeat §8, plus:
```
sudo iptables -S
sudo ip6tables -S
```
Expected:

- `wg-quick@wg0` is `active`.
- The base rules loaded from `rules.v4`/`rules.v6` are intact (as in
  [VPS-SETUP.md §7.7](VPS-SETUP.md#77-reboot-and-verify)).
- The four hook rules are on top of them, each once.
- IPv6 is unchanged: only the base rules, `FORWARD DROP`.

This proves that the base firewall (`iptables-persistent`) and the tunnel hooks come up
independently at boot and do not collide.

> `51820/udp` is also open in the IPv6 `INPUT` chain. That is fine: a client may reach
> the VPS over IPv6 as the outer transport, while the tunnel itself carries IPv4.

## 10. Changing hooks safely
Rules learned while building this setup:

- **Never run `netfilter-persistent save` while `wg0` is up.** The hook rules would be
  saved into the base firewall, loaded at boot and then added again by every `up`,
  leaving duplicates.
- **Stop the tunnel before editing a hook**: `sudo systemctl stop wg-quick@wg0` → edit
  `wg0.conf` → `sudo systemctl start wg-quick@wg0`. If the file is edited first, the new
  `PostDown` tries to delete a rule that is not the one loaded; it fails with
  `Bad rule`, `wg-quick` aborts the remaining `PostDown` lines, and the old rules stay in
  the kernel (orphaned).
- **A failed `PostUp` leaves rules behind.** If one `PostUp` line fails, `wg-quick`
  deletes the interface but does **not** run `PostDown`, so the rules already added by
  the earlier lines stay in the kernel.
- **Cleaning orphaned rules**: list them with the three `iptables -S` commands from §8,
  then delete each extra rule with `sudo iptables [-t <table>] -D <same specification>`.
  `-D` removes only the first matching rule, so with duplicates each command removes one
  copy. Then prove the symmetry again: `systemctl stop` → only the policies are left;
  `systemctl start` → each rule appears once.
- **`MASQUERADE` must use `-o <PUBLIC_IFACE>`, not `-o wg0`.** With `-o wg0`,
  internet-bound traffic leaves without NAT (with a private source address, so replies
  cannot come back), and traffic between peers is NATed instead.

---

## Current state
- `wg0` up with `10.10.0.1/24`, listening on `51820/udp`.
- Peers: the Raspberry Pi (`10.10.0.2` and the home LAN), added in
  [RASPBERRY-WIREGUARD.md §6](RASPBERRY-WIREGUARD.md#6-add-the-pi-as-a-peer-on-the-vps).
- Enabled at boot through `wg-quick@wg0`; survives a reboot alongside the base firewall.
- IPv4-only tunnel; IPv6 `FORWARD` stays at `DROP`.

Next: the clients (phone, laptop), in documents to come.

## Related documents
- [VPS-SETUP.md](VPS-SETUP.md) — SSH hardening and the base firewall this setup builds on.
- [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) — preparing the Raspberry Pi.
- [RASPBERRY-WIREGUARD.md](RASPBERRY-WIREGUARD.md) — the Pi as the first peer and home
  LAN gateway.
