# Raspberry Pi — host firewall (nftables table `inet pifw`)

Record of the host firewall on the Raspberry Pi: one nftables table of its own, loaded at
boot by a small systemd unit, that filters what reaches the Pi (`input`) and closes the
path from the home LAN into the tunnel (`forward`).

## Why a firewall on a Pi behind NAT
1. **NAT is not a firewall for the LAN.** The home router's NAT only stops unsolicited
   IPv4 connections from the internet. The home LAN itself is not a trusted zone: the TV,
   IoT devices and guests' phones are on it too.
2. **The Pi bridges the home LAN and the VPN.** With the `FORWARD` policy at `ACCEPT`, a
   LAN device only needs a route `10.10.0.0/24 via <PI_LAN_IP>` to start connections
   through the Pi into the tunnel. The VPS accepts that source, because the Pi peer's
   `AllowedIPs` include `<HOME_LAN_SUBNET>`; the clients accept it too, because their
   `AllowedIPs` include it (SPLIT) or `0.0.0.0/0` (FULL). The `FORWARD` rules from the
   `wg0.conf` hooks ([RASPBERRY-WIREGUARD.md §5.2](RASPBERRY-WIREGUARD.md#52-the-four-hooks))
   are `ACCEPT` rules: with policy `ACCEPT` they restrict nothing. Blocking needs a `DROP`
   policy or explicit drop rules.
3. **IPv6 has no NAT.** The Pi had global IPv6 addresses and `sshd` listened on
   `[::]:22`. Whether SSH was reachable from the internet over IPv6 depended only on the
   home router's IPv6 firewall.
4. **Future services listen on all interfaces.** Pi-hole and Docker containers will open
   ports on every interface unless something filters them (host-networked services
   through `input`; Docker bridge ports go through `forward`).

## Design
- **An `inet` table of its own, `pifw`.** An `inet` table covers IPv4 and IPv6 with one
  set of rules. It lives next to the `ip filter`, `ip nat` and `ip mangle` tables that the
  WireGuard `iptables` hooks create (and, later, Docker's), without touching them.
- **`input` policy `drop`.** Only what is listed is accepted.
- **`forward` policy `accept` plus two explicit drops.** Why not policy `drop`: in
  nftables a packet must be accepted by **every** base chain registered on the same hook.
  A `forward` chain with policy `drop` here would also drop the traffic that Docker's own
  tables accept, and Docker networking would break. Two targeted drop rules close exactly
  the paths that must be closed.
- **No `flush ruleset`.** It would wipe the WireGuard hook rules (and Docker's). The file
  replaces only its own table.

> **Prerequisites**:
> - [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) done: SSH hardened (key-only, port `22`).
> - [RASPBERRY-WIREGUARD.md](RASPBERRY-WIREGUARD.md) done: `wg0` up and enabled at boot.

Placeholders (`<HOME_LAN_SUBNET>`, `<PI_LAN_IP>`, `<WLAN_IFACE>`, ...) are defined in the
[README](../README.md#placeholders).

---

## 0. Inspect the starting point
```
nft --version
```
Prints the nftables version (`nftables v1.1.x` here). The syntax below needs nothing newer.

```
systemctl is-enabled nftables
cat /etc/nftables.conf
```
The first must print `disabled`. Debian's `/etc/nftables.conf` starts with
`flush ruleset` and defines an `inet filter` table with empty chains (`input`,
`forward`, `output`); if the `nftables` service were
enabled, it would load that file at boot and wipe every other rule. It must never be
enabled alongside this firewall.

```
sudo nft list tables
```
Lists the tables in the kernel. Here: `ip filter`, `ip nat` and `ip mangle`, created by the
WireGuard hooks.

```
sudo ss -tulpn
```
Lists every listening socket with its process (the options are explained in
[VPS-SETUP.md §7.1](VPS-SETUP.md#71-inventory-what-is-listening)). What was found:

- `sshd` on `0.0.0.0:22` and `[::]:22` → exposed on IPv4 and IPv6.
- `avahi-daemon` on UDP `5353` (and random ports) → mDNS, see below.
- NetworkManager's DHCPv6 client on UDP `546`, on a link-local address.
- A UDP port with no process → the WireGuard socket. It belongs to the kernel, not to a
  process, and its port is random at each boot (the Pi has no `ListenPort`).

```
sudo sshd -T | grep -i '^port'
```
Prints the port sshd really uses: `port 22`.

```
ip -6 addr show scope global
```
Shows the global IPv6 addresses. Here there were global addresses on the LAN interface
**and** on the Wi-Fi interface.

### 0.1 Is SSH reachable over IPv6 from the internet?
From **outside** the home network, for example from the VPS:
```
nc -6 -zv -w 5 <PI_PUBLIC_IPV6> 22
```
Tries a TCP connection to port `22` on the Pi's global IPv6 address (`-z` only checks
the port, `-v` prints the result, `-w 5` gives up after 5 seconds). Here it **timed out**:
the home router filters incoming IPv6. The Pi was not exposed, but only thanks to the
router.

> **Rule: test exposure from outside.** Running the same command on the Pi itself always
> succeeds: the Pi connects to itself and the packet never crosses the router. This
> mistake was made once during this rebuild.

### 0.2 Turn off Wi-Fi
The Pi was on the home LAN twice, by cable and by Wi-Fi. Two interfaces on the same
network make routing unpredictable and add a second way in; a `forward` rule that matched
only the LAN interface could also be bypassed through the Wi-Fi address.

```
sudo nmcli radio wifi off
```
Turns the Wi-Fi radio off. It needs `sudo`; NetworkManager remembers the state across
reboots.

```
nmcli radio wifi
ip -br addr show <WLAN_IFACE>
```
The first must print `disabled`; the second must show the interface `DOWN`, with no
address.

The first `forward` rule (§1) matches `iifname != "wg0"`, so it covers every non-tunnel
interface anyway, Wi-Fi included.

### 0.3 Disable avahi
`avahi-daemon` announces and resolves `.local` names (mDNS). It is not needed: the Pi is
reached by IP address.
```
sudo systemctl disable --now avahi-daemon.socket avahi-daemon.service
```
Stops both units and keeps them from starting at boot. The warning that "triggering units
are still active" is normal when the socket and the service are disabled at once.

```
systemctl is-active avahi-daemon.socket avahi-daemon.service
sudo ss -ulpn | grep 5353
```
The first must print `inactive` twice; the second must print nothing.

## 1. The rules file
```
sudo install -d -m 755 /etc/nftables.d
```
Creates `/etc/nftables.d` owned by `root`, if it does not exist.

```
sudo nano /etc/nftables.d/pifw.nft
```
Content:
```
#!/usr/sbin/nft -f
# Pi host firewall. Replaces only its own table; never flush the ruleset.
table inet pifw
delete table inet pifw

table inet pifw {
    set admin_nets {
        type ipv4_addr
        flags interval
        elements = { <HOME_LAN_SUBNET>, 10.10.0.0/24 }
    }

    chain input {
        type filter hook input priority filter; policy drop;

        iifname "lo" accept
        ct state established,related accept
        ct state invalid counter drop

        meta l4proto icmp accept
        meta l4proto ipv6-icmp accept

        udp sport 67 udp dport 68 accept
        udp sport 547 udp dport 546 accept

        tcp dport 22 ip saddr @admin_nets accept

        counter comment "dropped by policy"
    }

    chain forward {
        type filter hook forward priority filter; policy accept;

        iifname != "wg0" oifname "wg0" ct state new counter drop
        iifname "wg0" ip daddr != <HOME_LAN_SUBNET> counter drop
    }
}
```

| Block | What it does and why |
|---|---|
| `table inet pifw` / `delete table inet pifw` / `table inet pifw { ... }` | The first line creates the table empty if it does not exist, so `delete` never fails; the delete removes the old version; the block defines the new one. `nft -f` applies the whole file atomically: if anything is wrong, nothing changes. |
| `set admin_nets` | The networks allowed to administer the Pi: the home LAN (IPv4) and the VPN. `flags interval` lets the set hold ranges. Pi-hole's rules reuse it (§10). |
| `iifname "lo" accept` | Local programs talking to local services. |
| `ct state established,related accept` | Replies to everything the Pi starts: updates, DNS, NTP and the WireGuard tunnel itself. The Pi dials the VPS, and the keepalive keeps that conntrack entry alive, so the tunnel needs no open port. |
| `ct state invalid counter drop` | Packets that belong to no valid connection. |
| `meta l4proto icmp accept` / `ipv6-icmp accept` | `ping`, "packet too big" (Path MTU discovery) and IPv6 Neighbor Discovery, without which IPv6 does not work. |
| `udp sport 67 udp dport 68` / `udp sport 547 udp dport 546` | DHCP and DHCPv6 replies. IPv4 renewals are unicast and also match `established`; DHCPv6 replies answer a multicast request, which conntrack does not link to the reply, so that rule is required. |
| `tcp dport 22 ip saddr @admin_nets accept` | New SSH connections, only from the IPv4 home LAN and the VPN. SSH over IPv6 is therefore closed. |
| `counter comment "dropped by policy"` | Counts what the `drop` policy is about to drop. |
| forward rule 1: `iifname != "wg0" oifname "wg0" ct state new counter drop` | No **new** connection from any non-tunnel interface into `wg0`: a LAN device cannot use the Pi to reach the VPN. Replies to connections started from the tunnel still pass. The Pi's own traffic goes through `output`, not `forward`, so it is not affected. |
| forward rule 2: `iifname "wg0" ip daddr != <HOME_LAN_SUBNET> counter drop` | Tunnel traffic (IPv4) may only reach the home LAN: the Pi cannot be used as an exit to the internet. IPv6 never enters `wg0`: the hub peer's `AllowedIPs` on the Pi are IPv4 only, so WireGuard drops any inner IPv6 packet. |

`iifname`/`oifname` match the interface by name, so the rules load at boot even before
`wg0` exists.

### 1.1 Check before applying
```
grep -n '<' /etc/nftables.d/pifw.nft
```
Must print nothing: no placeholder is left in the file.

```
sudo nft -c -f /etc/nftables.d/pifw.nft && echo OK
```
`-c` checks the file without applying it; `OK` is printed only if the check passes.

> **Lesson learned: always check with `-c` first.** The file first contained only the
> `forward` chain, with a literal placeholder. `nft -c` rejected it
> (`unexpected '{'`) and nothing changed: a chain must live inside a table.

## 2. Load it with a safety net
```
sudo systemd-run --on-active=5m --unit=pifw-revert /usr/sbin/nft delete table inet pifw
```
Schedules the removal of the whole table in 5 minutes, in case a mistake locks SSH out. It
prints `Running timer as unit: pifw-revert.timer`. It is the same mechanism as on the VPS
([VPS-SETUP.md §7.2](VPS-SETUP.md#72-safety-net)).

```
sudo nft -f /etc/nftables.d/pifw.nft
```
Loads the table.

```
sudo nft list table inet pifw
```
Shows the set with both networks, both chains and the counters at `0`. The current SSH
session survives: it matches `established`.

## 3. Test within the 5 minutes
From a **new** terminal on the laptop:
```
ssh <USER>@<PI_LAN_IP>
```
Only a new connection proves the SSH rule; the open session only proves `established`.
If it fails, touch nothing and wait for the timer to remove the table. To re-arm it
afterwards, run `sudo systemctl stop pifw-revert.timer` first (see
[VPS-SETUP.md §7.2](VPS-SETUP.md#72-safety-net)).

On the Pi:
```
sudo wg show
```
Must show a recent handshake: the tunnel still works.

```
curl -4 -s ifconfig.me ; echo
```
Must print the home public IP: internet access works (`; echo` only adds the missing
newline).

```
getent hosts debian.org
```
Must print an address: DNS resolution works.

On the VPS:
```
ping -c 3 <HOME_ROUTER_IP>
```
Must answer, with TTL `63`: the path through the Pi into the home LAN still works.

The policy counter already showed 3 dropped packets: real traffic being blocked (the
cause was not investigated; typically multicast or announcements from LAN devices).

## 4. Cancel the timer
Only after every test in §3 has passed:
```
sudo systemctl stop pifw-revert.timer
systemctl list-timers | grep pifw
```
Cancels the timer; the second command must print nothing.

```
sudo nft list table inet pifw
```
The table must still be loaded.

> **Rule: test first, cancel after.** This time the timer was cancelled before the SSH
> test. It worked, but the safety net must exist while testing. On the VPS the timer
> caused the opposite problem; see the lesson in
> [VPS-SETUP.md §7.6](VPS-SETUP.md#76-persist-the-rules).

## 5. Load it at boot
```
systemctl is-enabled nftables
```
Must still print `disabled`. If it is enabled, `sudo systemctl disable nftables`, so its
`flush ruleset` never runs.

```
sudo nano /etc/systemd/system/pifw.service
```
Content:
```
[Unit]
Description=Pi host firewall (nft table inet pifw)
DefaultDependencies=no
Wants=network-pre.target
Before=network-pre.target shutdown.target
Conflicts=shutdown.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/nft -f /etc/nftables.d/pifw.nft
ExecStop=/usr/sbin/nft delete table inet pifw

[Install]
WantedBy=sysinit.target
```
- `Wants=` / `Before=network-pre.target` — the firewall is loaded before any network
  interface comes up, so there is no moment at boot without it. `Wants=` is needed
  because `network-pre.target` is a passive target that nothing pulls in by default:
  without it, the target may not be part of the boot and `Before=` would order nothing.
- `DefaultDependencies=no` — removes systemd's default ordering, which would otherwise
  start the unit too late to run before `network-pre.target`.
- `Before=shutdown.target` / `Conflicts=shutdown.target` — the unit is stopped cleanly at
  shutdown.
- `Type=oneshot` + `RemainAfterExit=yes` — `nft` runs once and exits; the unit then stays
  `active (exited)`, which shows the rules were loaded.
- `ExecStart` — loads the file from §1.
- `ExecStop` — removes only this table; the WireGuard and Docker rules are untouched.
- `WantedBy=sysinit.target` — started early in every boot.

```
sudo systemctl daemon-reload
sudo systemctl enable pifw
systemctl is-enabled pifw
```
`daemon-reload` makes systemd read the new unit; `enable` starts it at every boot; the last
command must print `enabled`. No `--now`: the rules are already loaded.

## 6. Reboot test
```
sudo reboot
```
Only a real reboot proves the boot behaviour. After it:

- A new `ssh` from the laptop works.
- `systemctl status pifw --no-pager` → `active (exited)`.
- `sudo nft list tables` → `inet pifw` next to `ip filter`, `ip nat` and `ip mangle`:
  nothing was flushed.
- `sudo wg show` → a fresh handshake (with a new random listening port, which is normal).
- `nmcli radio wifi` → `disabled`.
- `systemctl is-active avahi-daemon.socket` → `inactive`.
- On the VPS, `ping -c 3 <HOME_ROUTER_IP>` answers, with TTL `63`.

## 7. Prove the LAN→tunnel bridge is closed
The laptop plays the role of a compromised LAN device. At home, on Wi-Fi, with its VPN
off:
```
sudo ip route add 10.10.0.0/24 via <PI_LAN_IP>
```
Sends traffic for the VPN network to the Pi, as an attacker on the LAN would.

```
ping -c 3 -W 2 10.10.0.1
```
Must fail with `100% packet loss`: the Pi does not forward it into the tunnel.

```
sudo ip route del 10.10.0.0/24 via <PI_LAN_IP>
```
Removes the test route.

On the Pi:
```
sudo nft list chain inet pifw forward
```
The counter of forward rule 1 showed `3` packets, exactly the three pings: this rule is
what closes the path from the LAN into the tunnel.

## 8. If SSH is locked out after a reboot
The Pi is at home: connect a keyboard and a screen and run
```
sudo systemctl stop pifw
```
which removes only the `pifw` table (`ExecStop`), then fix the file.

## 9. Adding a service later
Add an `accept` rule to the `input` chain, limited to `@admin_nets`; check the file with
`nft -c` (§1.1); apply it with the revert timer (§2); test (§3); then cancel the timer
(§4). Pi-hole's rules are listed in §10.

This applies to services that listen on the Pi itself (host networking). A Docker port
published in bridge mode is translated (DNAT) to the container's address and goes through
`forward`, not `input`: the `input` rules do not filter it, and forward rule 2 drops it
when it comes from `wg0`. Such services need their own rules, documented with the
service.

## 10. Rules added later
The listing in §1 is the original file. Since then, these rules were added to the `input`
chain, right after the SSH rule:
```
        udp dport 53 ip saddr @admin_nets accept
        tcp dport 53 ip saddr @admin_nets accept
        tcp dport 8080 ip saddr @admin_nets accept
```
DNS and the web interface of Pi-hole, only from the home LAN and the VPN. They were added
following §9; see [PIHOLE-UNBOUND.md §4](PIHOLE-UNBOUND.md#4-firewall-rules).

---

## Current state
- Table `inet pifw` loaded at boot by `pifw.service`, reboot-verified; `nftables.service`
  disabled.
- `input`: policy `drop`; SSH only from `<HOME_LAN_SUBNET>` and `10.10.0.0/24`, so SSH over
  IPv6 is closed. DNS (`53`) and the Pi-hole web interface (`8080`) from the same
  networks (§10).
- `forward`: LAN→tunnel bridge closed (proved in §7); tunnel traffic limited to the home
  LAN.
- Wi-Fi off; avahi disabled.

Next: the other homelab services on the Pi.

Pending:

- Optional: reboot the VPS and confirm that the Pi reconnects on its own (keepalive).

## Related documents
- [RASPBERRY-WIREGUARD.md](RASPBERRY-WIREGUARD.md) — the Pi as WireGuard spoke and LAN
  gateway, whose hook rules live next to this table.
- [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) — SSH hardening of the Pi.
- [VPS-SETUP.md](VPS-SETUP.md) — the VPS firewall and the revert-timer lesson.
- [PIHOLE-UNBOUND.md](PIHOLE-UNBOUND.md) — Pi-hole and Unbound, whose DNS and web rules
  are in §10.
