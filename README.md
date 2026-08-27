# myVPN — Self-hosted WireGuard VPN with a VPS hub

A self-hosted, privacy-focused VPN built manually with **WireGuard**, using a cloud
**VPS as a hub** and a **Raspberry Pi** as a reverse tunnel and gateway to the home
LAN. No ports are opened on the home router.

This repository documents the whole process step by step and ships sanitized
configuration templates. It is a learning / portfolio project.

## Goals

- Access the **entire home LAN** from anywhere (NAS, other machines, etc.).
- **Exit to the internet through the VPS public IP**, switchable between full tunnel
  and split tunnel.
- **Zero open ports** on the home router: the Raspberry Pi initiates an outbound
  connection to the VPS and keeps it alive.
- Maximum control: everything is self-hosted, no third-party VPN coordinator.

## Architecture

```
                 INTERNET
                    │
       ┌────────────┴─────────────┐
       │      VPS (cloud hub)      │  static public IP
       │      wg0 = 10.10.0.1      │  listens on UDP 51820
       │  NAT/MASQUERADE + FORWARD │
       └────────────┬─────────────┘
                    ▲  outbound tunnel (no router ports opened)
                    │  PersistentKeepalive
       ┌────────────┴─────────────┐
       │   Raspberry Pi (home)     │
       │   wg0 = 10.10.0.2         │
       │   gateway → home LAN      │
       └────────────┬─────────────┘
                    │
              [ Home LAN ]

   Clients (phone 10.10.0.11, laptop 10.10.0.12) connect to the VPS hub.
```

Hub-and-spoke topology: all peer traffic flows through the VPS.

## Documentation

| Document | What it covers |
|---|---|
| [docs/PLAN.md](docs/PLAN.md) | Design and architecture: the phases and the reasoning behind each decision |
| [docs/SETUP.md](docs/SETUP.md) | Every command used to build it, with an explanation of what each one does |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | 10 real problems hit along the way: symptom, root cause, fix, lesson |
| [docs/OPERATIONS.md](docs/OPERATIONS.md) | Day-to-day usage: switching profiles, health checks, adding devices |
| [docs/HARDENING.md](docs/HARDENING.md) | VPS SSH hardening and firewall |

## Repository layout

```
.
├── README.md
├── docs/
│   ├── PLAN.md              # design and phases
│   ├── SETUP.md             # all commands, explained
│   ├── TROUBLESHOOTING.md   # real problems and fixes
│   ├── OPERATIONS.md        # daily usage
│   └── HARDENING.md         # SSH hardening and firewall
├── vps/
│   └── wg0.conf.example
├── raspberry/
│   └── wg0.conf.example
└── clients/
    ├── client.conf.example
    ├── laptop-split.conf.example
    ├── laptop-full.conf.example
    └── phone-notes.md
```

## Tech stack

- **WireGuard** (kernel-space, modern crypto, UDP).
- **iptables** for NAT/MASQUERADE and a default-DROP firewall.
- **systemd** (`wg-quick@wg0`) for auto start; `ssh.socket` for the SSH port.
- **fail2ban** + SSH hardening (key-only, non-default port).
- **openresolv** for DNS inside the tunnel on the Linux client.

## Phases

- **Phase 0** — Provision and harden the VPS. (done — see `docs/HARDENING.md`)
- **Phase 1** — VPS as WireGuard hub. (done)
- **Phase 2** — Raspberry Pi reverse tunnel + LAN gateway. (done)
- **Phase 3** — Clients (phone, laptop) with split/full profiles. (done)
- **Phase 4** — Verification and tests. (done)
- **Phase 5** — Robustness (own DNS resolver, auto-recovery, backups). (planned)

## Results

All the original goals are met and verified:

| Goal | Verified by |
|---|---|
| Reach the whole home LAN from outside | ping to a home device replies with `ttl=63`, proving it was routed through the Pi |
| Exit to the internet via the VPS IP | `curl -4 ifconfig.me` returns the VPS public IP with the FULL profile |
| No ports opened on the home router | the Pi keeps an outbound tunnel alive with `PersistentKeepalive` |
| Switchable split / full profiles | two profiles per client, one active at a time |
| Survives reboots | tunnels come back automatically on VPS, Pi and laptop |
| No IPv6 leaks in full tunnel | `curl -6 ifconfig.me` fails while the tunnel is up |


## Security notes

- **Private keys never leave their device** and are never committed. The `.gitignore`
  blocks `*.key` and real `*.conf` files; only `*.conf.example` templates are tracked.
- All configs here are **templates with placeholders** (`<VPS_PUBLIC_IP>`,
  `<HOME_LAN_SUBNET>`, `<...KEY>`), not real values.
- The VPS firewall uses a **default DROP** policy, allowing only SSH, WireGuard,
  loopback, established connections and ICMP.

## License

Personal learning project. Use at your own risk.
