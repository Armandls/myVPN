# myVPN — Self-hosted VPN and homelab

Personal homelab and WireGuard VPN project, being rebuilt step by step from a clean base.
For now the repository documents the preparation of the two hosts (the Raspberry Pi and
the VPS), the WireGuard hub on the VPS, the Pi as its first peer and home LAN gateway, and
the phone and laptop clients. Every command is listed with what it does and why.

## Documents

| Document | What it covers |
|---|---|
| [docs/RASPBERRY-SETUP.md](docs/RASPBERRY-SETUP.md) | Raspberry Pi OS Lite install, microSD → NVMe clone with `rpi-clone`, boot order, SSH hardening |
| [docs/VPS-SETUP.md](docs/VPS-SETUP.md) | VPS SSH hardening: key-only login, `AuthenticationMethods`, `DisableForwarding`, banner, custom port via `ssh.socket`, Fail2ban, base firewall (iptables IPv4/IPv6, `INPUT`/`FORWARD` DROP, `iptables-persistent`) |
| [docs/VPS-WIREGUARD.md](docs/VPS-WIREGUARD.md) | WireGuard hub on the VPS: IPv4-only design, persistent `ip_forward`, key pair, `wg0.conf` with `PostUp`/`PostDown` hooks (FORWARD, MASQUERADE, MSS clamping), `wg-quick@wg0` at boot, verification and safe hook changes |
| [docs/RASPBERRY-WIREGUARD.md](docs/RASPBERRY-WIREGUARD.md) | WireGuard on the Pi as spoke and home LAN gateway: `iptables` on Raspberry Pi OS, persistent `ip_forward`, key pair, `wg0.conf` dialling the hub (keepalive, FORWARD to the LAN, MASQUERADE, MSS clamping), Pi peer on the VPS, verification and reboot test |
| [docs/CLIENTS.md](docs/CLIENTS.md) | Phone and laptop clients: SPLIT and FULL profiles sharing one key, IPv6 captured with `::/0`, laptop setup on Arch Linux (`resolvconf` provider), phone profiles by QR code, client peers on the VPS (`reload` vs `stop`/`start`), tests and source address filtering |

## Placeholders

Every sensitive or machine-specific value is replaced by a placeholder. Real values are
never committed: the `.gitignore` blocks `*.conf`, `*.key` and `.env`.

| Placeholder | Meaning |
|---|---|
| `<USER>` | Login user on the Pi, set in Raspberry Pi Imager |
| `<PI_HOSTNAME>` | Hostname of the Pi, set in Raspberry Pi Imager |
| `<PI_LAN_IP>` | Pi address on the home LAN |
| `<LAN_IFACE>` | Name of the Pi network interface on the home LAN (the one holding `<PI_LAN_IP>`) |
| `<HOME_LAN_SUBNET>` | Home LAN subnet in CIDR notation (network address of `<PI_LAN_IP>` with the prefix shown by `ip -4 addr`, usually `/24`) |
| `<HOME_ROUTER_IP>` | Home router address on the home LAN |
| `<PI_PRIVATE_KEY>` | WireGuard private key of the Pi (content of `/etc/wireguard/privatekey` on the Pi, never leaves the Pi) |
| `<PI_PUBLIC_KEY>` | WireGuard public key of the Pi (content of `/etc/wireguard/publickey` on the Pi) |
| `<LAPTOP_PRIVATE_KEY>` | WireGuard private key of the laptop (content of `/etc/wireguard/laptop_private` on the laptop, never leaves the laptop) |
| `<LAPTOP_PUBLIC_KEY>` | WireGuard public key of the laptop (content of `/etc/wireguard/laptop_public` on the laptop) |
| `<PHONE_PRIVATE_KEY>` | WireGuard private key of the phone (generated on the laptop, imported by QR code, then deleted: lives only in the phone's WireGuard app) |
| `<PHONE_PUBLIC_KEY>` | WireGuard public key of the phone |
| `<ADMIN_USER>` | Admin user on the VPS (with sudo) |
| `<VPS_PUBLIC_IP>` | Public IPv4 of the VPS |
| `<SSH_PORT>` | Custom SSH port on the VPS |
| `<VPS_PRIVATE_KEY>` | WireGuard private key of the VPS (content of `/etc/wireguard/privatekey`, never leaves the VPS) |
| `<VPS_PUBLIC_KEY>` | WireGuard public key of the VPS (content of `/etc/wireguard/publickey` on the VPS) |
| `<PUBLIC_IFACE>` | Name of the VPS public network interface (the one holding `<VPS_PUBLIC_IP>`) |
| `<KEY_NAME>` | File name of the VPS SSH key pair on the laptop (`~/.ssh/<KEY_NAME>`) |
| `<ALIAS>` | Host alias for the VPS in the laptop's `~/.ssh/config` |
| `<comment>` | Free-text label stored in an SSH public key (`ssh-keygen -C`) |

## Status / next

The WireGuard hub on the VPS is up with three peers: the Pi, gateway to the home LAN, and
the phone and laptop clients, each with a SPLIT and a FULL profile. Next: Pi-hole and the
other homelab services on the Pi, documented here as they are rebuilt.

## License

[MIT](LICENSE).
