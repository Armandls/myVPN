# myVPN — Self-hosted VPN and homelab

Personal homelab and WireGuard VPN project, being rebuilt step by step from a clean base.
For now the repository documents the preparation of the two hosts (the Raspberry Pi and
the VPS) and the WireGuard hub on the VPS. Every command is listed with what it does and
why.

## Documents

| Document | What it covers |
|---|---|
| [docs/RASPBERRY-SETUP.md](docs/RASPBERRY-SETUP.md) | Raspberry Pi OS Lite install, microSD → NVMe clone with `rpi-clone`, boot order, SSH hardening |
| [docs/VPS-SETUP.md](docs/VPS-SETUP.md) | VPS SSH hardening: key-only login, `AuthenticationMethods`, `DisableForwarding`, banner, custom port via `ssh.socket`, Fail2ban, base firewall (iptables IPv4/IPv6, `INPUT`/`FORWARD` DROP, `iptables-persistent`) |
| [docs/VPS-WIREGUARD.md](docs/VPS-WIREGUARD.md) | WireGuard hub on the VPS (no peers yet): IPv4-only design, persistent `ip_forward`, key pair, `wg0.conf` with `PostUp`/`PostDown` hooks (FORWARD, MASQUERADE, MSS clamping), `wg-quick@wg0` at boot, verification and safe hook changes |

## Placeholders

Every sensitive or machine-specific value is replaced by a placeholder. Real values are
never committed: the `.gitignore` blocks `*.conf`, `*.key` and `.env`.

| Placeholder | Meaning |
|---|---|
| `<USER>` | Login user on the Pi, set in Raspberry Pi Imager |
| `<PI_HOSTNAME>` | Hostname of the Pi, set in Raspberry Pi Imager |
| `<PI_LAN_IP>` | Pi address on the home LAN |
| `<ADMIN_USER>` | Admin user on the VPS (with sudo) |
| `<VPS_PUBLIC_IP>` | Public IPv4 of the VPS |
| `<SSH_PORT>` | Custom SSH port on the VPS |
| `<VPS_PRIVATE_KEY>` | WireGuard private key of the VPS (content of `/etc/wireguard/privatekey`, never leaves the VPS) |
| `<PUBLIC_IFACE>` | Name of the VPS public network interface (the one holding `<VPS_PUBLIC_IP>`) |
| `<KEY_NAME>` | File name of the VPS SSH key pair on the laptop (`~/.ssh/<KEY_NAME>`) |
| `<ALIAS>` | Host alias for the VPS in the laptop's `~/.ssh/config` |
| `<comment>` | Free-text label stored in an SSH public key (`ssh-keygen -C`) |

## Status / next

The WireGuard hub on the VPS is up, with no peers yet. The Pi as the first peer and home
LAN gateway, the other clients and the homelab services on the Pi will be documented here
as they are rebuilt.

## License

[MIT](LICENSE).
