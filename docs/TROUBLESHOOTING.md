# Troubleshooting

Every real problem hit while building this VPN, with the exact error message, the root
cause, the fix, and the lesson. Indexed by symptom so it can be searched quickly.

| # | Symptom | Area |
|---|---|---|
| [1](#1-ssh-keygen-no-such-file-or-directory) | `Saving key ... failed: No such file or directory` | SSH keys |
| [2](#2-ssh-copy-id-no-identities-found) | `ERROR: No identities found` | SSH keys |
| [3](#3-passwordauthentication-stays-enabled) | Password login still works after setting `no` | SSH config |
| [4](#4-error-opening-terminal-xterm-kitty) | `Error opening terminal: xterm-kitty` | Remote editors |
| [5](#5-ssh-on-the-new-port-refuses-connections) | `Connection refused` on the new SSH port | systemd socket |
| [6](#6-key-is-not-the-correct-length-or-format) | `Key is not the correct length or format` | WireGuard config |
| [7](#7-iptables-command-not-found-on-the-pi) | `iptables: command not found` | Raspberry Pi OS |
| [8](#8-the-vps-cannot-reach-the-home-lan) | 100% packet loss to the home LAN | Routing |
| [9](#9-full-tunnel-fails-on-arch-three-chained-issues) | `resolvconf: signature mismatch` / `nft: No such file or directory` | Linux client |
| [10](#10-ipv6-left-disabled-after-a-failed-tunnel-start) | IPv6 stays off after a failed `wg-quick up` | Hooks |

---

## 1. ssh-keygen: No such file or directory

**Symptom**
```
Enter file in which to save the key (/home/user/.ssh/id_ed25519): ~/.ssh/ovh_vps
Saving key "~/.ssh/ovh_vps" failed: No such file or directory
```

**Root cause**
The `~` was typed **inside the interactive prompt**. Tilde expansion is done by the
shell, not by the program reading the prompt, so `ssh-keygen` treated `~` as a literal
directory name that does not exist.

**Fix**
Pass the path on the command line (where the shell expands it), or type an absolute
path at the prompt:
```bash
ssh-keygen -t ed25519 -a 100 -C "vps-key" -f ~/.ssh/<KEY_NAME>
```

**Lesson**
`~` only expands when the shell parses it. Inside prompts, use absolute paths.

---

## 2. ssh-copy-id: No identities found

**Symptom**
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed:
/usr/bin/ssh-copy-id: ERROR: No identities found
```

**Root cause**
With no key specified, `ssh-copy-id` looks for default names (`id_rsa`,
`id_ed25519`...). The key had a custom name, so nothing was found.

**Fix**
```bash
ssh-copy-id -i ~/.ssh/<KEY_NAME>.pub <USER>@<VPS_PUBLIC_IP>
```

**Lesson**
Custom key names must be passed explicitly with `-i`, both to `ssh-copy-id` and to
`ssh` (or configured via `IdentityFile` in `~/.ssh/config`).

---

## 3. PasswordAuthentication stays enabled

**Symptom**
`/etc/ssh/sshd_config.d/` contained two files with contradictory values:
```
50-cloud-init.conf        → PasswordAuthentication yes
60-cloudimg-settings.conf → PasswordAuthentication no
```
and the effective value was `yes`.

**Root cause**
Drop-in files are read in lexicographic order, and for most directives **sshd keeps
the first value it sees**. `50-` is read before `60-`, so the `yes` won. Adding a
`99-hardening.conf` with `no` would not have helped either — it loads even later.

**Fix**
Remove both provider files, add an explicit hardening drop-in, and stop cloud-init
from regenerating one:
```bash
# /etc/cloud/cloud.cfg.d/99-disable-ssh-config.cfg
ssh_pwauth: false
```
```bash
# /etc/ssh/sshd_config.d/99-hardening.conf
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
```
Always verify the **effective** configuration rather than reading files:
```bash
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
```

**Lesson**
In sshd, first match wins — the opposite of many other config systems. Verify with
`sshd -T`, and remember cloud images may recreate their own drop-ins.

---

## 4. Error opening terminal: xterm-kitty

**Symptom**
```
sudo nano /etc/wireguard/wg0.conf
Error opening terminal: xterm-kitty.
```
Also seen as `ncurses: cannot initialize terminal type ($TERM="xterm-kitty")`.

**Root cause**
The local terminal (kitty) sets `TERM=xterm-kitty`, but the remote host has no
terminfo entry for it, so ncurses-based programs cannot initialize.

**Fix — quick, one-off**
```bash
sudo TERM=xterm-256color nano /path/to/file
```

**Fix — permanent, system-wide**
```bash
# from the local machine
infocmp -x xterm-kitty | ssh <HOST> 'sudo tic -x -'
```
`tic` prints a harmless warning about the description field; it still installs.

Note that installing it only for your user is not enough: `sudo nano` runs as root and
needs the entry in the system terminfo database (`/usr/share/terminfo/`).

**Lesson**
Terminal emulators with non-standard `TERM` values need their terminfo shipped to every
remote host, installed system-wide so root sees it too.

---

## 5. SSH on the new port refuses connections

**Symptom**
After changing the SSH port via a `ssh.socket` override, the server was listening but
clients got:
```
ssh: connect to host <VPS_PUBLIC_IP> port <SSH_PORT>: Connection refused
```
`ss` showed only:
```
LISTEN 0 4096 [::]:<SSH_PORT> [::]:*
```

**Root cause**
The override contained a single `ListenStream=<SSH_PORT>`, which systemd bound to
IPv6 only. Connections arriving over IPv4 had nothing listening, hence *refused*
(not a timeout, which would suggest a firewall).

**Fix**
Declare both families explicitly:
```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:<SSH_PORT>
ListenStream=[::]:<SSH_PORT>
```
The empty `ListenStream=` resets the inherited port 22. Then:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo ss -tlnp | grep <SSH_PORT>     # two lines expected
```

**Lesson**
`Connection refused` means something answered "nobody is listening" — a socket/bind
problem. A firewall drop would instead produce a timeout. On recent Ubuntu the SSH port
lives in `ssh.socket`, not `sshd_config`.

---

## 6. Key is not the correct length or format

**Symptom A** — leftover editor keystrokes
```
wg setconf wg0 /dev/fd/63
Key is not the correct length or format: `<44-char-key>=wq'
Configuration parsing error
```

**Root cause A**
The `wq` at the end is a vim save-and-quit command that was typed **into the buffer**
instead of being executed, corrupting the key. A valid WireGuard key is 44 base64
characters ending in `=`, with nothing after it.

**Symptom B** — unexpanded shell variable
```
Key is not the correct length or format: `$PRIV'
```

**Root cause B**
The config was generated with a heredoc using `\$PRIV`. Inside an **unquoted** heredoc
(`<<EOF`) the shell already expands variables, so escaping it wrote the literal text
`$PRIV` into the file.

**Fix**
Use `$PRIV` (no backslash) and verify the file afterwards:
```bash
sudo grep PrivateKey /etc/wireguard/<name>.conf | cut -d' ' -f3 | wg pubkey
```
This derives the public key from whatever is in the file; it must match the public key
registered on the peer. It also validates the format in one step.

**Lesson**
Always verify a key by deriving its public counterpart rather than eyeballing it.
`<<EOF` expands variables; `<<'EOF'` does not.

---

## 7. iptables: command not found on the Pi

**Symptom**
```
[#] iptables -t nat -A POSTROUTING -o <LAN_IFACE> -j MASQUERADE
/usr/bin/wg-quick: line 295: iptables: command not found
[#] ip link delete dev wg0
```
`wg-quick` treats a failing `PostUp` as fatal and tears the interface down.

**Root cause**
Recent Raspberry Pi OS uses nftables and does not ship the `iptables` binary, which
`wg-quick` invokes for the NAT rule.

**Fix**
```bash
sudo apt install iptables
```
This provides `iptables-nft` (iptables syntax translated to nftables), fully
compatible with the rules used here and consistent with the VPS configuration.

**Lesson**
A failing hook aborts the whole tunnel. Distros are migrating to nftables, so do not
assume `iptables` exists.

---

## 8. The VPS cannot reach the home LAN

**Symptom**
The tunnel was healthy (handshake in both directions, `ping 10.10.0.2` fine) but:
```
ping -c 3 <HOME_ROUTER_IP>
3 packets transmitted, 0 received, 100% packet loss
```
Meanwhile on the VPS:
```
ip route | grep <HOME_LAN_SUBNET>
(empty)
```

**Root cause**
The Pi peer had been added with `wg set`. That registers the peer and its
`allowed-ips` inside WireGuard, but **does not create the kernel route** to the LAN
subnet. Without a route the VPS had no idea it should send that traffic to `wg0`.

**Fix — quick test**
```bash
sudo ip route add <HOME_LAN_SUBNET> dev wg0
```

**Fix — persistent (preferred)**
Keep the `[Peer]` block in `wg0.conf` and let `wg-quick` build everything:
```bash
sudo systemctl restart wg-quick@wg0
ip route | grep <HOME_LAN_SUBNET>     # now shows 'dev wg0'
```
`wg-quick` derives routes from each peer's `AllowedIPs`, and they are recreated on
every boot.

**Lesson**
`wg set` configures WireGuard; `wg-quick` configures WireGuard **and** the routing
table. For anything involving routed subnets, use `wg-quick`.

Diagnostic tip: a reply with `ttl=63` instead of `64` proves the packet crossed one
extra hop, confirming it was routed through the Pi.

---

## 9. Full tunnel fails on Arch: three chained issues

Bringing up `AllowedIPs = 0.0.0.0/0` on the laptop failed three times for three
different reasons.

### 9a. resolvconf missing

**Symptom**
```
[#] resolvconf -a casa-full -m 0 -x
/usr/bin/wg-quick: line 32: resolvconf: command not found
[#] ip link delete dev casa-full
```

**Root cause**
The `DNS =` directive makes `wg-quick` call `resolvconf`, which was not installed.

**Fix**
```bash
sudo pacman -S openresolv
```

### 9b. Kernel modules out of sync

**Symptom**
```
[#] nft -f /dev/fd/63
/dev/fd/63:5:82-95: Error: Could not process rule: No such file or directory
```
Repeated for the rules using `fib` and `ct mark`.

**Root cause**
A full tunnel makes `wg-quick` add extra nftables rules (kill-switch) that need the
`nft_fib_ipv4` and `nft_ct` kernel modules. Diagnosis:
```bash
uname -r                  # 7.1.8-arch1-3   ← running kernel
ls /usr/lib/modules/      # 7.1.9-arch1-2   ← only the new modules on disk
```
The kernel had been upgraded without rebooting. Already-loaded modules kept working,
but no **new** module could be loaded, so nftables could not instantiate those rules.

**Fix**
```bash
sudo reboot
```

**Lesson**
On rolling-release distros, a kernel upgrade without a reboot silently breaks loading
of any module not already in memory. `No such file or directory` from `nft` usually
means a missing kernel module, not a missing file.

### 9c. NetworkManager overwriting resolv.conf

**Symptom**
```
[#] resolvconf -a casa-full -m 0 -x
resolvconf: signature mismatch: /etc/resolv.conf
resolvconf: run `resolvconf -u` to update
```
And `/etc/resolv.conf` starting with `# Generated by NetworkManager`.

**Root cause**
NetworkManager was writing `/etc/resolv.conf` itself, so openresolv found a file
without its signature and refused to touch it. Setting only `dns=resolvconf` was not
enough: it selects the DNS backend but leaves NetworkManager as the file's owner
(`rc-manager=symlink` by default). The setting appeared to work until the next reboot,
when NetworkManager recreated the file.

**Fix**
```ini
# /etc/NetworkManager/conf.d/rc-manager.conf
[main]
dns=resolvconf
rc-manager=resolvconf
```
```bash
sudo systemctl restart NetworkManager
sudo resolvconf -u
cat /etc/resolv.conf      # must read "Generated by resolvconf"
```
Verify it **survives a reboot** — that is where the incomplete version failed.

Rollback if DNS breaks:
```bash
sudo rm /etc/NetworkManager/conf.d/rc-manager.conf
sudo systemctl restart NetworkManager
```

**Lesson**
`dns=` and `rc-manager=` are different knobs; both are required. Any change to DNS
management must be validated across a reboot, not just right after applying it.

---

## 10. IPv6 left disabled after a failed tunnel start

**Symptom**
After a failed `wg-quick up`, IPv6 stayed off system-wide:
```bash
sysctl net.ipv6.conf.all.disable_ipv6
net.ipv6.conf.all.disable_ipv6 = 1
```

**Root cause**
IPv6 was being disabled from `PreUp` to prevent leaks in full-tunnel mode. When
`wg-quick` failed later (DNS, nftables...), it aborted before ever running `PostDown`,
so the restore step never executed.

**Fix**
Move the blocking hooks from `PreUp` to `PostUp`, so IPv6 is only touched once
everything else has succeeded:
```ini
PostUp   = sysctl -w net.ipv6.conf.all.disable_ipv6=1
PostUp   = sysctl -w net.ipv6.conf.default.disable_ipv6=1
PostDown = sysctl -w net.ipv6.conf.all.disable_ipv6=0
PostDown = sysctl -w net.ipv6.conf.default.disable_ipv6=0
```
Manual recovery if it happens:
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0
```

**Lesson**
Hooks that change global system state should run as late as possible, so a partial
failure cannot leave the system in a modified state. `PreUp` changes stick around if
the tunnel never comes up.

---

## General debugging checklist

```bash
sudo wg show                          # handshake, transfer, allowed ips
ip addr show <iface>                  # is the interface up with the right address
ip route                              # are the expected routes present
sudo iptables -S INPUT                # firewall rules and default policy
sudo iptables -t nat -L POSTROUTING -n
sysctl net.ipv4.ip_forward            # forwarding enabled
cat /etc/resolv.conf                  # who owns DNS right now
curl -4 ifconfig.me                   # which IP the world sees
```

Reading the signals:
- **No handshake** → connectivity or key mismatch: check `Endpoint`, UDP 51820 open,
  and that each side has the other's correct public key.
- **Handshake but no traffic** → routing or NAT: check `AllowedIPs`, `ip route`,
  `MASQUERADE`, and `ip_forward`.
- **`Connection refused`** → nothing listening (socket/bind).
- **Timeout / 100% loss** → dropped by firewall or no route.
- **`ttl` one lower than expected** → traffic crossed an extra hop (working as
  intended when routing through a gateway).
