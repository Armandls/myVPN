# Setup guide — all commands

Chronological record of every command used to build the VPN, with an explanation of
what each one does and why. All sensitive values are placeholders.

## Placeholders used

| Placeholder | Meaning |
|---|---|
| `<VPS_PUBLIC_IP>` | Public IPv4 of the VPS |
| `<SSH_PORT>` | Custom SSH port on the VPS |
| `<PUBLIC_IFACE>` | VPS interface facing the internet (e.g. `ens3`) |
| `<LAN_IFACE>` | Pi interface facing the home LAN (e.g. `eth0`) |
| `<HOME_LAN_SUBNET>` | Home LAN subnet (e.g. `192.168.x.0/24`) |
| `<HOME_ROUTER_IP>` | Home router / gateway IP |
| `<PI_LAN_IP>` | Pi address on the home LAN |
| `<PI_MAC>` | MAC address of the Pi LAN interface |
| `<VPS_PUBLIC_KEY>` | WireGuard public key of the VPS |
| `<PI_PUBLIC_KEY>` | WireGuard public key of the Pi |
| `<PHONE_PUBLIC_KEY>` | WireGuard public key of the phone |
| `<LAPTOP_PUBLIC_KEY>` | WireGuard public key of the laptop |

## VPN address map

| Device | VPN address | Role |
|---|---|---|
| VPS | `10.10.0.1` | Hub (only one with a static public IP) |
| Raspberry Pi | `10.10.0.2` | Reverse tunnel + LAN gateway |
| Phone | `10.10.0.11` | Roaming client |
| Laptop | `10.10.0.12` | Roaming client |

---

# Phase 1 — VPS as WireGuard hub

SSH hardening is documented separately in [HARDENING.md](HARDENING.md).

## 1.1 Install WireGuard

```bash
sudo apt install wireguard
```
Installs `wireguard-tools`, which provides the `wg` and `wg-quick` commands. The
WireGuard implementation itself lives in the kernel on modern Linux.

Verify:
```bash
wg --version
which wg-quick
```

## 1.2 Enable IP forwarding

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-wireguard-forward.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward     # must return 1
```

Why: by default the kernel drops packets that are not addressed to itself. IP
forwarding tells it to **route** them instead. Without this the VPS would receive
client traffic destined for the internet (or the home LAN) and silently discard it.

Why a file in `/etc/sysctl.d/`: values written directly to `/proc/sys/` are lost on
reboot. Files here are re-applied at every boot. The `99-` prefix means it loads last,
so it wins over provider defaults. `sysctl --system` applies everything immediately
without rebooting.

## 1.3 Generate the VPS key pair

```bash
cd /etc/wireguard
umask 077
wg genkey | sudo tee server_private.key
sudo cat server_private.key | wg pubkey | sudo tee server_public.key
```

- `umask 077` makes new files `600` (root-only). WireGuard refuses to start if the
  private key is world-readable.
- `wg genkey` produces a random private key (base64, 44 chars).
- `wg pubkey` derives the public key from the private one. This is a one-way
  operation: you can go private → public, never the reverse.

Rule to remember: the **private** key never leaves the machine; the **public** key is
what you copy into the other peer's `[Peer]` block.

## 1.4 Identify the public interface

```bash
ip route show default
ip -br addr
```
The `dev <name>` field of the default route is the interface facing the internet. It
is needed for the NAT rule (`MASQUERADE`) so traffic can exit through the VPS IP.

## 1.5 Create `/etc/wireguard/wg0.conf`

See [../vps/wg0.conf.example](../vps/wg0.conf.example) for the full template.

Key points:
- `Address = 10.10.0.1/24` — the VPS address *inside* the VPN (not its public IP).
- `ListenPort = 51820` — UDP port. The VPS is the only peer with a fixed, known port,
  because it is the only one with a stable public address.
- `PostUp`/`PostDown` — commands `wg-quick` runs right after bringing the interface
  up / down. Used here to add and remove firewall and NAT rules automatically.

The three `PostUp` rules:
```
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -j ACCEPT
iptables -t nat -A POSTROUTING -o <PUBLIC_IFACE> -j MASQUERADE
```
- The two `FORWARD` rules **authorize** traffic to traverse the VPS (IP forwarding
  enables the capability; the firewall must also allow it).
- `MASQUERADE` rewrites the source address of outgoing packets to the VPS public IP.
  Without it a client packet would leave with a private source (`10.10.0.11`) that the
  internet cannot route back — traffic would go out but replies would never return.

Each `PostDown` mirrors its `PostUp` with `-D` (delete) instead of `-A` (add), so no
stale rules are left behind when the tunnel goes down.

## 1.6 Firewall: default-DROP policy

Base security rules, persisted independently of WireGuard:

```bash
sudo apt install iptables-persistent -y

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport <SSH_PORT> -j ACCEPT
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Only after verifying SSH still works from ANOTHER terminal:
sudo iptables -P INPUT DROP

sudo netfilter-persistent save
```

Explanation of each rule:
- `-i lo` — loopback. Local services talk to themselves over `127.0.0.1`; blocking
  this breaks things.
- `ESTABLISHED,RELATED` — the most important rule. Allows **replies** to connections
  the VPS itself initiated (`apt update`, DNS...). Without it, all outbound traffic
  appears to break.
- SSH and WireGuard ports — the two services that must be reachable.
- `icmp echo-request` — lets the VPS answer pings, useful for diagnostics.
- `-P INPUT DROP` — flips the default from "allow anything" to "deny unless allowed".

Order is critical: add every ACCEPT rule and confirm SSH still works **before**
switching the policy to DROP, otherwise you lock yourself out. Keep a second SSH
session open as a safety net.

`INPUT` vs `FORWARD`: `INPUT` is traffic addressed *to* the VPS (this section);
`FORWARD` is traffic passing *through* it (handled in `wg0.conf`).

Why `iptables-persistent` and not `wg0.conf`: SSH access must not depend on the VPN
being up. If these rules lived in `PostUp` and `wg0` failed to start, you would end up
with `DROP` active and no SSH rule.

## 1.7 Bring the interface up and enable at boot

```bash
sudo wg-quick up wg0
sudo wg show
sudo systemctl enable wg-quick@wg0
```

`wg-quick up` automates what the WireGuard quick-start does manually: creates the
interface, loads the key and port, assigns the address, brings it up, adds peer routes
and runs the `PostUp` hooks.

`systemctl enable wg-quick@wg0` makes it come back automatically after a reboot.
`wg-quick@` is a systemd template unit; the part after `@` selects the config name.

Verify:
```bash
sudo wg show          # interface, public key, listening port
ip addr show wg0      # 10.10.0.1/24
sudo iptables -t nat -L POSTROUTING -n
```
`state UNKNOWN` on a WireGuard interface is normal, not an error.

---

# Phase 2 — Raspberry Pi (reverse tunnel + LAN gateway)

The Pi plays two roles: a WireGuard **client** that keeps an outbound tunnel to the
VPS (so no router ports need opening), and a **gateway** that forwards traffic into
the home LAN.

## 2.0 Network preparation

Get the MAC address of the LAN interface, then create a **DHCP reservation** on the
router so the Pi always gets the same address:
```bash
ip link show <LAN_IFACE>      # read the link/ether field
```
A gateway must have a stable address; if DHCP moved it, routing would break.

Disable Wi-Fi so there is a single interface and a single default route:
```bash
sudo nmcli radio wifi off
nmcli general status          # WIFI must read 'disabled'
ip route | grep default       # only the LAN interface should remain
```

## 2.1 Install WireGuard and iptables

```bash
sudo apt install wireguard
sudo apt install iptables
```
Recent Raspberry Pi OS ships nftables and does **not** include the `iptables` binary,
which `wg-quick` needs for the `PostUp` rule. Installing it provides `iptables-nft`
(iptables syntax on top of nftables), fully compatible with the rules used here.

## 2.2 Enable IP forwarding

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-wireguard-forward.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward
```
Same reason as on the VPS, but here it lets the Pi forward remote-client traffic into
the home LAN.

## 2.3 Generate the Pi key pair

```bash
sudo sh -c 'cd /etc/wireguard && umask 077 && wg genkey | tee pi_private.key | wg pubkey > pi_public.key'
sudo ls -l /etc/wireguard/      # both files must be -rw-------
sudo cat /etc/wireguard/pi_public.key
```

## 2.4 Create `/etc/wireguard/wg0.conf` on the Pi

See [../raspberry/wg0.conf.example](../raspberry/wg0.conf.example).

Key points:
- `Address = 10.10.0.2/24`.
- No `ListenPort`: the Pi is a client and uses a random source port. Only the hub
  needs a fixed port.
- `Endpoint = <VPS_PUBLIC_IP>:51820` — the Pi knows where the VPS is, so **the Pi
  initiates the connection**. This is the reverse tunnel: the connection is outbound,
  so no port forwarding is required on the home router.
- `AllowedIPs = 10.10.0.0/24` — VPN traffic is routed to the VPS.
- `PersistentKeepalive = 25` — sends a packet every 25 s. NAT devices forget idle
  mappings; without keepalive the router would drop the mapping once the tunnel went
  quiet and the VPS could no longer reach the Pi.
- `PostUp = iptables -t nat -A POSTROUTING -o <LAN_IFACE> -j MASQUERADE` — rewrites
  the source of packets entering the LAN to the Pi's LAN address, so home devices know
  where to send replies. This is what makes the whole LAN reachable.

## 2.5 Add the Pi peer on the VPS

Append to the VPS `/etc/wireguard/wg0.conf`:
```ini
[Peer]
# Raspberry Pi
PublicKey = <PI_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32, <HOME_LAN_SUBNET>
```

`AllowedIPs` here acts as a routing table: it tells the VPS "to reach the Pi *and the
home LAN*, send traffic to this peer".

Apply it by restarting the interface so `wg-quick` also creates the kernel route:
```bash
sudo systemctl restart wg-quick@wg0
ip route | grep <HOME_LAN_SUBNET>     # 'dev wg0' must appear
```

Do not use `wg set` for this: it registers the peer but does not create the route.
See [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## 2.6 Bring up and verify

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
sudo wg show                      # 'latest handshake' must be recent
```

End-to-end checks:
```bash
# from the Pi
ping -c 3 10.10.0.1               # the VPS over the VPN

# from the VPS
ping -c 3 10.10.0.2               # the Pi over the VPN
ping -c 3 <HOME_ROUTER_IP>        # a device on the home LAN, through the Pi
```
The LAN ping replying with `ttl=63` instead of `64` confirms the packet took one extra
hop, i.e. it was routed through the Pi.

Persistence check — reboot the Pi and confirm the tunnel comes back on its own:
```bash
sudo reboot
sudo wg show
```
The client listening port changes after each restart; that is expected, since the VPS
learns the endpoint dynamically from the handshake.

---

# Phase 3 — Clients

Keys are generated **on each device**, so no private key ever travels.

Every client needs two things done: its public key registered as a `[Peer]` on the
VPS, and a local config pointing at the VPS endpoint.

## 3.1 Register a client peer on the VPS

Append to the VPS `/etc/wireguard/wg0.conf`:
```ini
[Peer]
# Phone
PublicKey = <PHONE_PUBLIC_KEY>
AllowedIPs = 10.10.0.11/32
```
Only the client's own VPN address is listed: a client is an endpoint, not a gateway,
so nothing is routed *through* it.

Apply in place without dropping existing tunnels:
```bash
sudo wg set wg0 peer <PHONE_PUBLIC_KEY> allowed-ips 10.10.0.11/32
sudo wg show
```
Because the block is also in the file, it survives reboots.

## 3.2 Phone

See [../clients/phone-notes.md](../clients/phone-notes.md).

## 3.3 Laptop (Arch Linux)

```bash
sudo pacman -S wireguard-tools
sudo pacman -S openresolv        # provides resolvconf, needed by the DNS directive
```

Generate the key pair on the laptop:
```bash
sudo sh -c 'cd /etc/wireguard && umask 077 && wg genkey | tee laptop_private.key | wg pubkey > laptop_public.key'
sudo cat /etc/wireguard/laptop_public.key
```

Create the two profiles (see the templates):
- [../clients/laptop-split.conf.example](../clients/laptop-split.conf.example)
- [../clients/laptop-full.conf.example](../clients/laptop-full.conf.example)

A safe way to write the file without ever printing the private key:
```bash
sudo bash -c 'PRIV=$(cat /etc/wireguard/laptop_private.key); cat > /etc/wireguard/casa-split.conf <<EOF
[Interface]
Address = 10.10.0.12/32
PrivateKey = $PRIV
...
EOF
chmod 600 /etc/wireguard/casa-split.conf'
```
Inside an unquoted heredoc (`<<EOF`) the shell **does** expand variables, so write
`$PRIV`, not `\$PRIV`.

Confirm the config key matches the peer registered on the VPS:
```bash
sudo grep PrivateKey /etc/wireguard/casa-split.conf | cut -d' ' -f3 | wg pubkey
# must print <LAPTOP_PUBLIC_KEY>
```

DNS on Arch with NetworkManager — required for the `DNS =` directive to work:
```bash
# /etc/NetworkManager/conf.d/rc-manager.conf
[main]
dns=resolvconf
rc-manager=resolvconf
```
```bash
sudo systemctl restart NetworkManager
sudo resolvconf -u
cat /etc/resolv.conf     # header must read "Generated by resolvconf"
```
`dns=` selects the backend; `rc-manager=` decides who owns `/etc/resolv.conf`. Both
are needed, otherwise NetworkManager keeps rewriting the file and `resolvconf` fails
with a signature mismatch.

Bring a profile up:
```bash
sudo wg-quick up casa-split
sudo wg show
ping -c 3 10.10.0.1
sudo wg-quick down casa-split
```

---

# Verification

| Goal | Command | Expected result |
|---|---|---|
| Tunnel established | `sudo wg show` | recent `latest handshake` |
| Reach the VPS | `ping 10.10.0.1` | 0% packet loss |
| Reach the Pi | `ping 10.10.0.2` | 0% packet loss |
| Reach the home LAN | `ping <HOME_ROUTER_IP>` | 0% loss, `ttl=63` |
| Exit through the VPS | `curl -4 ifconfig.me` | prints `<VPS_PUBLIC_IP>` |
| No IPv6 leak (full) | `curl -6 ifconfig.me` | fails to connect |
| DNS from the tunnel | `cat /etc/resolv.conf` | the DNS set in the profile |
| Survives reboot | `sudo reboot` then `wg show` | tunnel back automatically |
