# Placeholders

Every sensitive or machine-specific value in this repository is replaced by a
placeholder. This table defines all of them; each document links here instead of
keeping its own list.

## Network

| Placeholder | Meaning |
|---|---|
| `<VPS_PUBLIC_IP>` | Public IPv4 of the VPS |
| `<PUBLIC_IFACE>` | VPS interface facing the internet (e.g. `ens3`) |
| `<LAN_IFACE>` | Pi interface facing the home LAN (e.g. `eth0`) |
| `<HOME_LAN_SUBNET>` | Home LAN subnet (e.g. `192.168.x.0/24`) |
| `<HOME_ROUTER_IP>` | Home router / gateway IP |
| `<PI_LAN_IP>` | Pi address on the home LAN |

## SSH and accounts

| Placeholder | Meaning |
|---|---|
| `<SSH_PORT>` | Custom SSH port on the VPS |
| `<ADMIN_USER>` | Admin user on the VPS (with sudo) |
| `<KEY_NAME>` | File name of the SSH key pair on the laptop (`~/.ssh/<KEY_NAME>`) |
| `<ALIAS>` | Host alias for the VPS in the laptop's `~/.ssh/config` |
| `<PI_HOSTNAME>` | Hostname of the Pi, set in Raspberry Pi Imager |
| `<USER>` | Login user on the Pi, set in Raspberry Pi Imager |

## WireGuard keys

Private keys appear only in the `*.conf.example` templates, as the slot where the real
key goes. They never leave the device that generated them.

| Placeholder | Meaning |
|---|---|
| `<VPS_PUBLIC_KEY>` / `<VPS_PRIVATE_KEY>` | WireGuard key pair of the VPS |
| `<PI_PUBLIC_KEY>` / `<PI_PRIVATE_KEY>` | WireGuard key pair of the Pi |
| `<PHONE_PUBLIC_KEY>` | WireGuard public key of the phone |
| `<LAPTOP_PUBLIC_KEY>` / `<LAPTOP_PRIVATE_KEY>` | WireGuard key pair of the laptop |
