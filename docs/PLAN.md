# Plan: Manual WireGuard VPN with a VPS as hub

## Goals
- Access the **whole home LAN** from outside (not just a single device).
- **Exit to the internet through the VPS IP**, switchable (full tunnel / split tunnel).
- **No open ports** on the home router (the Raspberry Pi initiates an outbound
  connection to the VPS).
- Clients: phone (QR) and laptop.

## Devices
- **VPS** (cloud): hub with a static public IP.
- **Raspberry Pi 5** (home): reverse tunnel + gateway to the LAN.
- **Clients**: phone and laptop.

---

## Key concepts (the "under the hood")
WireGuard uses key pairs (public/private), like SSH. Each peer knows the others by
their **public key**.

- **`AllowedIPs`**: defines WHICH traffic is routed to a peer. It acts both as a
  routing table and as a filter. The most important concept to master.
- **`Endpoint`**: the public IP:port where a peer can be reached. Only the VPS has a
  fixed endpoint (that is why it is the hub).
- **`PersistentKeepalive`**: packets every 25s to keep the outbound connection alive
  through the router's NAT (key to the reverse tunnel).
- **Forwarding + NAT (iptables)**: so traffic can "pass through" the VPS and the Pi
  toward other networks.

---

## Topology

```
Internal WireGuard network: 10.10.0.0/24

[Phone]      10.10.0.11 ─┐
[Laptop]     10.10.0.12 ─┤
                         │  Endpoint: <VPS_PUBLIC_IP>:51820
                         ▼
[VPS] 10.10.0.1  ◄── HUB, the only one with a static public IP
                         ▲
                         │  Pi initiates the OUTBOUND connection
                         │  + PersistentKeepalive
[Raspberry Pi] 10.10.0.2 ┘
      │
      └─► forwards to home LAN (<HOME_LAN_SUBNET>)
```

Hub-and-spoke topology: all peer traffic passes through the VPS.

---

## Phase 0 — Provision the VPS
- **Minimum specs**: 1 vCPU, 1 GB RAM, ~10-20 GB disk.
- **What matters**: good bandwidth / monthly traffic (full tunnel = all your
  internet goes through here) and low latency (VPS close to you).
- **OS**: Debian 12 or Ubuntu LTS.
- Requirements: a dedicated public IPv4 + SSH access.

## Phase 1 — Prepare the VPS (hub)
1. SSH hardening (see HARDENING.md).
2. Install WireGuard (`wireguard-tools`).
3. Enable IP forwarding (`net.ipv4.ip_forward=1`).
4. Generate the VPS key pair.
5. Create `/etc/wireguard/wg0.conf`:
   - Interface `wg0`, IP `10.10.0.1/24`, `ListenPort = 51820`.
   - iptables rules in `PostUp/PostDown`: FORWARD + **MASQUERADE** (NAT) on the
     public interface → enables "exit through the VPS IP".
   - One `[Peer]` block per device.
6. Firewall: allow **51820/udp** (WireGuard) + the SSH port.
7. `wg-quick up wg0` + `systemctl enable wg-quick@wg0` (auto start).

## Phase 2 — Configure the Raspberry Pi (reverse tunnel + LAN gateway)
1. Install WireGuard.
2. Enable IP forwarding (to forward toward the home LAN).
3. Generate the Pi key pair.
4. Create `/etc/wireguard/wg0.conf`:
   - IP `10.10.0.2/24`.
   - `[Peer]` = VPS:
     - `Endpoint = <VPS_PUBLIC_IP>:51820`.
     - `AllowedIPs = 10.10.0.0/24`.
     - **`PersistentKeepalive = 25`** (keeps the tunnel alive through NAT).
   - iptables `PostUp/PostDown`: **MASQUERADE** toward the home LAN.
5. On the VPS, add the Pi `[Peer]` with:
   `AllowedIPs = 10.10.0.2/32, <HOME_LAN_SUBNET>`
   (tells the VPS: to reach the home LAN, send traffic to the Pi).
6. Bring up and enable at boot.
7. **Intermediate test**: from the VPS `ping 10.10.0.2` then ping a LAN device.

> The home LAN must be **different** from the network you connect from (avoid route
> conflicts, e.g. a hotel Wi-Fi on the same subnet).

## Phase 3 — Clients (phone and laptop)
1. Generate a key pair for each client.
2. Add its `[Peer]` on the VPS (`AllowedIPs = 10.10.0.11/32`, etc.).
3. Two profiles per client (the "switchable" part):
   - **SPLIT** (home LAN only): `AllowedIPs = 10.10.0.0/24, <HOME_LAN_SUBNET>`.
   - **FULL** (everything through the VPS): `AllowedIPs = 0.0.0.0/0`.
   - Both with `Endpoint = <VPS_PUBLIC_IP>:51820` and optional `DNS`.
4. **Phone**: generate a QR code per profile for the WireGuard app.
5. **Laptop**: import the `.conf` file.

## Phase 4 — Verification and tests
1. **Handshake**: `wg show` with a recent "latest handshake" on each device.
2. **LAN access**: from the phone (on mobile data) ping/SSH the Pi and another home
   device.
3. **Exit through VPS**: with the FULL profile, confirm the public IP is the VPS one.
4. **Switching**: split vs full changes behavior.
5. **Reboots**: reboot Pi and VPS; the tunnel re-establishes on its own.

## Phase 5 — Robustness (optional, recommended)
- **DNS**: own resolver or Cloudflare/Quad9 in the profiles.
- **Auto-recovery**: systemd timer on the Pi that restarts `wg0` if the handshake is
  lost for too long.
- **Documentation**: keep an IP map, public keys (NEVER private ones) and useful
  commands.
- **Backup** of `.conf` files (without exposing private keys).

---

## What you learn
Key pairs, the hub-and-spoke model, `AllowedIPs` as routes+filter, kernel IP
forwarding, NAT/MASQUERADE with iptables, keepalive through NAT, split vs full
tunnel, and debugging with `wg show` / `tcpdump`.
