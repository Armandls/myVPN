# Raspberry Pi — WireGuard spoke and home LAN gateway

Record of the WireGuard setup on the Raspberry Pi, the first peer of the VPN, and of
adding it as a peer on the VPS hub.

The Pi (`10.10.0.2`) is a WireGuard **client** of the hub (`10.10.0.1`) and the **door to
the home LAN**. Remote clients reach the home LAN through the VPS and the Pi:
```
phone / laptop
   │  encrypted, over UDP to <VPS_PUBLIC_IP>:51820
   ▼
VPS wg0 (10.10.0.1)  ── hub
   │  tunnel (the Pi dialled out to the VPS, keepalive every 25 s)
   ▼
Raspberry Pi wg0 (10.10.0.2)
   │  FORWARD + MASQUERADE out through <LAN_IFACE>
   ▼
home LAN (<HOME_LAN_SUBNET>)
```

- **The Pi dials out.** It opens the tunnel towards the VPS and keeps it alive with a
  packet every 25 seconds, so the home router needs **no open ports** and no port
  forwarding. The Pi has no `ListenPort`.
- **The Pi is the gateway to the home LAN.** It forwards tunnel traffic to the LAN and
  rewrites its source address to the Pi's own LAN address (`MASQUERADE`), so LAN devices
  reply to the Pi as if it were talking to them itself. Nothing is changed on the home
  router.
- **The VPS learns where the LAN is from the Pi peer's `AllowedIPs`**: the Pi peer on the
  VPS lists `<HOME_LAN_SUBNET>`, so the VPS routes that range into the tunnel, towards
  the Pi.
- **IPv4 only**, consistent with the hub (see the
  [design decision in VPS-WIREGUARD.md](VPS-WIREGUARD.md#design-decision-ipv4-only-tunnel)).

> **Prerequisites**:
> - [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) done: Raspberry Pi OS on NVMe, SSH hardened.
> - [VPS-WIREGUARD.md](VPS-WIREGUARD.md) done: the hub is up on `10.10.0.1/24`,
>   listening on `51820/udp`, enabled at boot.

Placeholders (`<LAN_IFACE>`, `<HOME_LAN_SUBNET>`, ...) are defined in the
[README](../README.md#placeholders).

---

## 0. Check that the Pi is clean
Before installing anything, confirm there is no leftover tunnel or firewall state.

```
ip link show wg0
```
Must print `Device "wg0" does not exist.`: no WireGuard interface is up.

```
ls -la /etc/wireguard
```
Must not contain old keys or configurations (before WireGuard is installed, the
directory may not exist at all).

```
sudo nft list ruleset
```
Lists every firewall rule in the kernel (Raspberry Pi OS uses nftables). It must be
empty: Docker is not installed yet, so nothing has added rules.

## 1. Install WireGuard and `iptables`
```
sudo apt update
sudo apt install wireguard iptables
```
`apt update` refreshes the package lists; `apt install` installs `wireguard` (which pulls
in `wireguard-tools`, the `wg` and `wg-quick` commands) and `iptables`.

Why `iptables`: Raspberry Pi OS ships only nftables (`nft`), without the `iptables`
command. The hooks below use the same `iptables` syntax as on the VPS, which keeps both
configurations easy to compare. Underneath, the rules still go into nftables.

```
iptables -V
```
Must print `iptables v1.8.x (nf_tables)`: the `nf_tables` part confirms that `iptables`
is only a front end and the rules are stored in nftables.

> `iptables-persistent` is **not** installed on the Pi: there is no base firewall to
> persist yet. The only rules are the tunnel hooks, which `wg-quick` adds and removes.

## 2. Persistent IPv4 forwarding
The Pi must route packets between the tunnel and the home LAN. Without IPv4 forwarding,
the kernel drops every packet that arrives through `wg0` for a LAN device.

Debian 13 has no `/etc/sysctl.conf`, so the setting goes in a file of its own, the same
name as on the VPS:
```
sudo nano /etc/sysctl.d/99-vpn.conf
```
Content:
```
# file: /etc/sysctl.d/99-vpn.conf
# The Pi is the gateway to the home LAN: route packets between wg0 and the LAN
net.ipv4.ip_forward = 1
```

Apply and verify:
```
sudo sysctl --system | grep -A1 99-vpn
```
Re-applies every sysctl file and keeps only the lines about this one. The output must
show `* Applying /etc/sysctl.d/99-vpn.conf ...` followed by `net.ipv4.ip_forward = 1`:
the file is read and the value is set. Because the setting is in a file, it survives a
reboot (checked in §10).

## 3. Key pair
```
sudo sh -c 'cd /etc/wireguard && umask 077 && wg genkey | tee privatekey | wg pubkey > publickey'
```
Generates the Pi key pair as `root`, inside `/etc/wireguard`, with permissions `600`. It
is the same command as on the VPS; every part is explained in
[VPS-WIREGUARD.md §3](VPS-WIREGUARD.md#3-key-pair).

```
sudo ls -l /etc/wireguard/
```
Every file must show `-rw------- ... root root`.

```
sudo cat /etc/wireguard/publickey
```
Prints the Pi public key, `<PI_PUBLIC_KEY>`. It goes into the Pi peer on the VPS (§6).

On the **VPS**, the same command prints the VPS public key, `<VPS_PUBLIC_KEY>`. It goes
into the `[Peer]` section of the Pi configuration (§5).

The Pi **private key** never leaves the Pi.

## 4. Identify the LAN interface and subnet
```
ip route show default
```
Shows the default route, towards the home router. The word after `dev` is the name of
the Pi's LAN interface: it is `<LAN_IFACE>` in the configuration below.

```
ip -4 addr show <LAN_IFACE>
```
Shows the IPv4 address of that interface, as `inet <PI_LAN_IP>/<prefix>` (usually
`/24`). The network address of that range (the address with the host part set to `0`,
written with that prefix) is the home LAN subnet, `<HOME_LAN_SUBNET>`.

**Recommended (pending)**: create a DHCP reservation for the Pi in the home router, so
`<PI_LAN_IP>` never changes. The tunnel works without it, because the hooks use the
interface and the subnet, not the Pi's address. It will be needed later, when Pi-hole
serves DNS on the Pi.

> **Limitation**: if a remote network (a café, a hotel) uses the same range as the home
> LAN, home devices are unreachable from there. The client's route to its own local
> network wins over the route through the tunnel.

## 5. Configuration: `/etc/wireguard/wg0.conf`
```
sudo nano /etc/wireguard/wg0.conf
```
Opens the tunnel configuration as `root`.

Content:
```
[Interface]
Address = 10.10.0.2/24
PrivateKey = <PI_PRIVATE_KEY>

PostUp = iptables -A FORWARD -i %i -o <LAN_IFACE> -d <HOME_LAN_SUBNET> -j ACCEPT
PostUp = iptables -A FORWARD -i <LAN_IFACE> -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o <LAN_IFACE> -j MASQUERADE
PostUp = iptables -t mangle -A FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

PostDown = iptables -D FORWARD -i %i -o <LAN_IFACE> -d <HOME_LAN_SUBNET> -j ACCEPT
PostDown = iptables -D FORWARD -i <LAN_IFACE> -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o <LAN_IFACE> -j MASQUERADE
PostDown = iptables -t mangle -D FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

[Peer]
# VPS (hub)
PublicKey = <VPS_PUBLIC_KEY>
Endpoint = <VPS_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.0/24
PersistentKeepalive = 25
```
`<PI_PRIVATE_KEY>` is the content of `/etc/wireguard/privatekey` on the Pi. Write every
line with single spaces around `=`: the key check in §7 depends on it.

```
sudo chmod 600 /etc/wireguard/wg0.conf
```
Read and write for `root` only: the file contains the private key. `nano` creates new
files as `644`.

### 5.1 Differences from the hub
- **No `ListenPort`**: the Pi always starts the connection, so it does not need a fixed
  port. It uses a random source port, different at each boot, and the home router's NAT
  takes care of sending the replies back to it.
- **`Endpoint = <VPS_PUBLIC_IP>:51820`**: where to find the hub. Only the Pi side has an
  endpoint; the VPS learns the Pi's address from its packets (§6).
- **`PersistentKeepalive = 25`**: the Pi sends a small packet every 25 seconds, even with
  no traffic. This keeps the home router's NAT mapping open, so the VPS can reach the Pi
  at any moment, not only right after the Pi has sent something.
- **`AllowedIPs = 10.10.0.0/24`**: on a client, this decides two things. Outgoing, only
  VPN addresses are routed into the tunnel, so the Pi's own internet traffic keeps going
  directly through the home router. Incoming, only packets with a VPN source address are
  accepted from the hub.

### 5.2 The four hooks
The hooks work exactly as on the VPS (`%i` = `wg0`, `-A` in `PostUp` mirrored by `-D` in
`PostDown`, `filter`/`nat`/`mangle` tables); see
[VPS-WIREGUARD.md §5.2](VPS-WIREGUARD.md#52-how-the-hooks-work). What changes is the
direction: here traffic goes from the tunnel to the LAN.

**Rule 1 — `FORWARD -i %i -o <LAN_IFACE> -d <HOME_LAN_SUBNET> -j ACCEPT`.** Lets traffic
from the tunnel be routed to the home LAN, and only to it (`-d <HOME_LAN_SUBNET>`). Once
the `FORWARD` policy is `DROP` (Docker or the planned Pi firewall), the Pi cannot be used
as an exit to the internet: a packet from the tunnel to any other destination matches no
rule and is dropped. Today, with policy `ACCEPT`, what prevents it is the VPS, which only
routes `10.10.0.2` and `<HOME_LAN_SUBNET>` to the Pi.

**Rule 2 — `FORWARD -i <LAN_IFACE> -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j
ACCEPT`.** Lets LAN traffic into the tunnel only if it belongs to a connection that a
VPN peer already started. Replies come back; once the policy is `DROP`, LAN devices
cannot start a connection into the tunnel.

**Rule 3 — `-t nat POSTROUTING -s 10.10.0.0/24 -o <LAN_IFACE> -j MASQUERADE`.** LAN
devices use the home router as their default route, not the Pi. Without NAT, a LAN device
would send its reply to `10.10.0.x` to the home router, which knows nothing about the VPN,
and the reply would be lost. `MASQUERADE` rewrites the source of tunnel traffic to the
Pi's LAN address, so the LAN device replies to the Pi; connection tracking then reverses
the rewrite and sends the reply into the tunnel. The home router needs no extra route.

**Rule 4 — `-t mangle FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS
--clamp-mss-to-pmtu`.** On the VPS this rule is mostly a safety net; here it really
matters. LAN devices do not know about the tunnel: they have MTU `1500` and announce an
MSS of `1460`. The rule lowers the MSS in their `SYN-ACK` packets going into the tunnel to
`1420 − 40` = `1380`, so remote clients never send them TCP segments too big for the
tunnel. The MSS is explained in detail in
[VPS-WIREGUARD.md §5.3](VPS-WIREGUARD.md#53-the-four-rules).

**Why explicit `FORWARD` rules?** Today the Pi's `FORWARD` policy is `ACCEPT`, so
forwarding would work without rules 1 and 2. They are there because Docker, installed
later for Pi-hole, may set the `FORWARD` policy to `DROP` (and the planned Pi firewall
will): without these rules the tunnel would then stop reaching the LAN. Once the policy is
`DROP`, they are also what limits forwarding to the cases above.

> Every `PostDown` mirrors its `PostUp`. Replace the placeholders with the real values in
> the file on the Pi: a literal `<LAN_IFACE>` breaks the hooks (see the note at the end of
> [VPS-WIREGUARD.md §5.5](VPS-WIREGUARD.md#55-what-is-deliberately-not-there)).

## 6. Add the Pi as a peer on the VPS
These steps run **on the VPS**.

```
sudo systemctl stop wg-quick@wg0
```
Takes the hub down before editing its configuration. This is the safe habit from
[VPS-WIREGUARD.md §10](VPS-WIREGUARD.md#10-changing-hooks-safely): the file is never
edited while the tunnel it describes is up.

```
sudo nano /etc/wireguard/wg0.conf
```
Append at the end of the VPS configuration:
```
[Peer]
# Raspberry Pi (home gateway)
PublicKey = <PI_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32, <HOME_LAN_SUBNET>
```

```
sudo systemctl start wg-quick@wg0
```
Brings the hub back up with the new peer and runs its hooks again.

- `PublicKey` — the Pi public key from §3: the VPS accepts only packets encrypted by the
  Pi's private key for this peer.
- `AllowedIPs = 10.10.0.2/32, <HOME_LAN_SUBNET>` — `10.10.0.2/32` is exactly the Pi.
  `<HOME_LAN_SUBNET>` tells the VPS that every address in the home LAN is behind the Pi:
  `wg-quick` adds a route for that range through `wg0`, and the VPS sends traffic for the
  home LAN to this peer.
- **No `Endpoint` and no `PersistentKeepalive`** on the VPS side: the VPS learns the Pi's
  address and port from its first packet, and follows them if the home public IP
  changes. The keepalive is the Pi's job.

**Why `stop`/`start` and not `systemctl reload`?** `reload` (which runs
`wg syncconf`) updates the peers without dropping the tunnel, but it does **not** add
routes. The VPS would know the Pi peer but would not route `<HOME_LAN_SUBNET>` into `wg0`.
`stop`/`start` runs the whole `wg-quick` setup, routes included.

## 7. Pre-flight checks on the Pi
Before bringing the tunnel up for the first time.

```
sudo grep PrivateKey /etc/wireguard/wg0.conf | cut -d' ' -f3 | wg pubkey
sudo cat /etc/wireguard/publickey
```
The first command derives the public key from the private key pasted in `wg0.conf`; the
second prints the saved public key. Both outputs must be identical: the right key is in
the configuration.

```
sudo ls -l /etc/wireguard/
```
Every file, `wg0.conf` included, must be `-rw------- root root`.

The `PostUp`/`PostDown` symmetry is checked through systemd after the reboot (§10).

## 8. Start at boot with systemd
```
sudo systemctl enable --now wg-quick@wg0
```
`enable` starts the tunnel at every boot; `--now` also starts it right away. As on the
VPS, `wg-quick@wg0` uses `/etc/wireguard/wg0.conf`.

From now on, manage the tunnel with `sudo systemctl stop|start|restart wg-quick@wg0`, not
with `wg-quick` directly, so systemd stays in sync with the real state of the tunnel.

## 9. Verify end to end
### 9.1 On the Pi
```
sudo wg show
```
Must show the VPS peer with:

- `latest handshake: N seconds ago` — the two sides have exchanged keys.
- `transfer:` with values that grow over time.
- `persistent keepalive: every 25 seconds`.
- `listening port:` with a random number. This is normal: with no `ListenPort`, the
  kernel picks one.

If there is no handshake, the usual causes are a wrong key (on either side), a wrong
`Endpoint`, or `51820/udp` blocked on the way to the VPS.

```
ping -c 3 10.10.0.1
```
The hub must answer, in a few tens of milliseconds.

### 9.2 On the VPS
```
sudo wg show
```
Must show the Pi peer with a recent handshake, an `endpoint` made of the home public IP
and a random port, and `allowed ips: 10.10.0.2/32, <HOME_LAN_SUBNET>`.

```
ip route | grep wg0
```
Must show two routes:

- `10.10.0.0/24 dev wg0 proto kernel ...` — the VPN network, from `Address`.
- `<HOME_LAN_SUBNET> dev wg0 scope link` — the home LAN, added by `wg-quick` from the
  Pi peer's `AllowedIPs`.

```
ping -c 3 10.10.0.2
ping -c 3 <HOME_ROUTER_IP>
```
Both must answer: first the Pi itself, then the home router, a device on the home LAN.
With a router whose initial TTL is 64 (as most Linux-based routers), the router's replies
have a TTL one lower than the Pi's (`63` instead of `64`): the reply crossed one extra
hop, the Pi. That is the proof that the traffic went through the Pi gateway.

### 9.3 Back on the Pi
```
sudo iptables -t nat -L POSTROUTING -v -n
sudo iptables -L FORWARD -v -n
```
`-v` shows packet and byte counters for each rule; `-n` prints addresses as numbers. After
the pings from the VPS, the counters must be non-zero on the `MASQUERADE` rule and on both
`FORWARD` rules: rule 1 counts the ping requests going to the LAN, rule 2 the replies
coming back. The `MASQUERADE` counter grows by only one per connection (the three pings
count as `1`): the `nat` table only sees the first packet; conntrack applies the same
rewrite to the rest.

## 10. Reboot test
```
sudo reboot
```
Run on the Pi. Only a real reboot proves the boot behaviour. After reconnecting, repeat
§9.1: the tunnel must come up on its own, with a fresh handshake, and `ping 10.10.0.1`
must answer.

- The `listening port` is different after each boot, because there is no `ListenPort`.
  This is normal: the VPS learns the new port from the Pi's first packet.
- The first ping after boot may take a few hundred milliseconds; the next ones settle
  at the usual time.

Then the symmetry check, this time through systemd:
```
sudo systemctl stop wg-quick@wg0
```
Takes the tunnel down through systemd and runs the `PostDown` lines. Then list the hook
chains:
```
sudo iptables -S FORWARD
sudo iptables -t nat -S POSTROUTING
sudo iptables -t mangle -S FORWARD
```
They must show only the policies: `-P FORWARD ACCEPT`, `-P POSTROUTING ACCEPT` and
`-P FORWARD ACCEPT`.

```
sudo systemctl start wg-quick@wg0
```
The same three commands must show the four hook rules again, each once: two in
`filter FORWARD`, one in `nat POSTROUTING`, one in `mangle FORWARD`. Every `PostUp` has a
matching `PostDown`.

**Optional check**: reboot the VPS and confirm that the Pi reconnects on its own, thanks
to the keepalive, without touching the Pi.

---

## Current state
- Pi `wg0` up with `10.10.0.2/24`, dialling the hub with a keepalive every 25 seconds;
  enabled at boot through `wg-quick@wg0`.
- The Pi forwards tunnel traffic to the home LAN with `MASQUERADE`; nothing changed on the
  home router.
- The VPS knows the Pi peer and routes `<HOME_LAN_SUBNET>` through `wg0`.

Pending:

- DHCP reservation for the Pi in the home router (§4).
- A host firewall on the Pi: its `INPUT` and `FORWARD` policies are `ACCEPT` today. It is
  planned as its own nftables table.
- The clients (phone, laptop) and Pi-hole, in later documents.

## Related documents
- [VPS-WIREGUARD.md](VPS-WIREGUARD.md) — the WireGuard hub on the VPS, the hook rules and
  how to change hooks safely.
- [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) — preparing the Pi: OS on NVMe and SSH
  hardening.
