# VPS setup — SSH hardening and firewall

Record of the hardening steps applied to the VPS before setting up WireGuard.

> The base firewall is documented in §7. WireGuard is documented separately in
> [VPS-WIREGUARD.md](VPS-WIREGUARD.md).

## Server details
- **Provider**: cloud VPS. This build used 2 vCore, 4 GB RAM, 40 GB NVMe and
  unmetered traffic.
- **OS**: Ubuntu LTS.
- **Public IP (IPv4)**: `<VPS_PUBLIC_IP>`
- **Admin user**: `<ADMIN_USER>` (with sudo).

---

## 1. System update
```
sudo apt update && sudo apt upgrade -y
```
(May install a new kernel → reboot required.)

## 2. SSH key (on the local machine)
Generate the ed25519 key pair with a passphrase:
```
ssh-keygen -t ed25519 -C "<comment>" -f ~/.ssh/<KEY_NAME>
```
Creates a new key pair dedicated to the VPS, on the laptop.

- `-t ed25519` selects the Ed25519 algorithm: modern, secure and with short keys.
- `-C "<comment>"` is only a label stored in the public key, to recognise it later (for
  example in `authorized_keys`). It has no security role.
- `-f ~/.ssh/<KEY_NAME>` chooses the file name: the private key is `~/.ssh/<KEY_NAME>`
  (permissions 600) and the public key `~/.ssh/<KEY_NAME>.pub` (permissions 644).

`ssh-keygen` then asks for a **passphrase** at an interactive prompt. The passphrase
encrypts the private key on disk, so a stolen copy of the file is useless without it.
Type it at the prompt rather than passing it with `-N "..."`: anything written on the
command line is saved in the shell history.

Optionally, `-a <rounds>` (e.g. `-a 100`) raises the number of KDF rounds that protect
the passphrase (default 16), making brute-forcing a stolen key slower. It can also be
applied to an existing key with `ssh-keygen -p -a 100 -f ~/.ssh/<KEY_NAME>`.

> Note: the initial "No such file or directory" error came from typing `~` inside the
> interactive prompt (the shell does not expand the tilde there). Fix: use an absolute
> path or `-f ~/...` on the command line.

Copy the public key to the VPS:
```
ssh-copy-id -i ~/.ssh/<KEY_NAME>.pub <ADMIN_USER>@<VPS_PUBLIC_IP>
```
> The "No identities found" error was because the key has a non-standard name; fix by
> passing it with `-i`.

### Local SSH alias (~/.ssh/config)
```
Host <ALIAS>
    HostName <VPS_PUBLIC_IP>
    User <ADMIN_USER>
    Port <SSH_PORT>
    IdentityFile ~/.ssh/<KEY_NAME>
```
Connect: `ssh <ALIAS>`

## 3. Disable password and root login
Initial state in `/etc/ssh/sshd_config.d/`:
- `50-cloud-init.conf` → `PasswordAuthentication yes`
- `60-cloudimg-settings.conf` → `PasswordAuthentication no`

Since SSH applies the FIRST match (50 before 60), the effective value was `yes`.
Both were removed and a custom file created + cloud-init neutralized.

Neutralize cloud-init:
```
# file: /etc/cloud/cloud.cfg.d/99-disable-ssh-config.cfg
ssh_pwauth: false
```

Custom hardening file, now the only file in `/etc/ssh/sshd_config.d/`:
```
# file: /etc/ssh/sshd_config.d/99-hardening.conf
# Only <ADMIN_USER> may log in over SSH
AllowUsers <ADMIN_USER>

# Key-only login: no password authentication
PubkeyAuthentication yes
PasswordAuthentication no
AuthenticationMethods publickey

# No root login
PermitRootLogin no

# No keyboard-interactive authentication
KbdInteractiveAuthentication no

# Disable forwarding
DisableForwarding yes

# Only ~/.ssh/authorized_keys (drop the legacy authorized_keys2)
AuthorizedKeysFile .ssh/authorized_keys

# Pre-login notice
Banner /etc/ssh/banner
```
It is exactly the same set of directives as on the Pi. In short:

- `AllowUsers <ADMIN_USER>` — only this account may log in over SSH.
- `PubkeyAuthentication yes` / `PasswordAuthentication no` — key login allowed, password
  login rejected.
- `AuthenticationMethods publickey` — every login must succeed with a key; a second,
  independent lock in case another file re-enabled a password method.
- `PermitRootLogin no` — `root` cannot log in; administration goes through
  `<ADMIN_USER>` and `sudo`.
- `KbdInteractiveAuthentication no` — closes the other (PAM) way of asking for a
  password, pinned here so no later file can turn it back on.
- `DisableForwarding yes` — turns off all SSH forwarding (`ssh -L`/`-R`/`-D`, agent, X11,
  Unix sockets).
- `AuthorizedKeysFile .ssh/authorized_keys` — keys are read only from that file, not
  from the legacy `authorized_keys2`, leaving a single place to audit.
- `Banner /etc/ssh/banner` — text sent to anyone who connects, **before**
  authentication: an authorised-use notice that must not reveal anything useful
  (hostname, OS, provider, versions). It lives in `/etc/ssh/`, owned by `root`, not in
  the user's home, because sshd reads it as `root` before authentication and follows
  symlinks: a file in the home could be swapped for a symlink to a root-only file (e.g.
  `/etc/shadow` or a WireGuard private key), whose content sshd would then show to
  anyone who connects.

The full reasoning for each directive is in
[RASPBERRY-SETUP.md Step 11.4](RASPBERRY-SETUP.md#114-create-the-drop-in-file).

> `DisableForwarding` only concerns **SSH** forwarding. It has nothing to do with the
> kernel's `net.ipv4.ip_forward`, which the VPS needs as the WireGuard hub to route
> traffic between peers and out to the internet. WireGuard routing is not affected.

Restrict the file's ownership and permissions:
```
sudo chown root:root /etc/ssh/sshd_config.d/99-hardening.conf
sudo chmod 600 /etc/ssh/sshd_config.d/99-hardening.conf
```
Makes the file owned by `root` and readable/writable only by `root`, so no other local
user can modify it or read it (for example to learn which account is in `AllowUsers`).
`ls -la /etc/ssh/sshd_config.d/` then shows
`-rw------- 1 root root ... 99-hardening.conf`. `sshd -t` and `sshd -T` always need
`sudo` anyway: sshd must read the host private keys (`/etc/ssh/ssh_host_*_key`, readable
only by `root`), and without root it exits with `no hostkeys available`. With the drop-in
at `600`, the drop-in itself is also unreadable, so `Permission denied` on it is the
first error reported.

Create the banner referenced by `Banner`:
```
sudo nano /etc/ssh/banner
```
Opens a new file as `root`, so it is created owned by `root`. Write a short
authorised-use notice and save, for example:
```
Authorized access only. All connections may be monitored and logged.
```

```
sudo chown root:root /etc/ssh/banner
sudo chmod 644 /etc/ssh/banner
```
Makes the banner owned by `root`, world-readable like the other files in `/etc/ssh/`,
and writable only by `root`. The `chown` matters if the file was created or moved there
by the normal user: otherwise that user could still replace its content.

Apply and verify:
```
sudo sshd -t
```
Checks the whole SSH configuration for errors without applying it; it prints nothing
when everything is correct. Run it **before** restarting, so sshd is never restarted
with a broken configuration.

```
sudo systemctl restart ssh
```
Restarts the SSH server so it reads the new configuration. Existing sessions are not
dropped: keep the current one open until the tests below pass, as the way back if
something is wrong.

```
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication|permitrootlogin|allowusers|kbdinteractive|authenticationmethods|disableforwarding|authorizedkeysfile|banner'
```
`-T` prints the final configuration sshd actually uses, after merging every file.
Expected output (the order of the lines may differ):
```
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
disableforwarding yes
banner /etc/ssh/banner
authorizedkeysfile .ssh/authorized_keys
allowusers <ADMIN_USER>
authenticationmethods publickey
```
The full `sshd -T` output may still list `x11forwarding`, `allowtcpforwarding`,
`allowagentforwarding` and `allowstreamlocalforwarding` as `yes`. This is expected:
`disableforwarding yes` overrides them, so they have no effect.

Test from a **new** terminal on the laptop. Once §4 is done, use the alias
`ssh <ALIAS>` (or pass `-p <SSH_PORT>`); before §4 the port is still `22`. On recent
Ubuntu the listening port is set through `ssh.socket` (§4):
```
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive <ALIAS>
```
Tries to log in using both password methods, ignoring keys. It must fail with
`Permission denied (publickey)`: the text in parentheses is the list of methods the
server offers, and here it offers only `publickey`.

```
ssh -o IdentitiesOnly=yes <ALIAS>
```
Logs in with the key from the alias (asking for its passphrase). `IdentitiesOnly=yes`
makes ssh offer only the alias's `IdentityFile`, not other keys loaded in an agent, so
the test proves that this specific key works. It must succeed. Before the passphrase
prompt, the text of `/etc/ssh/banner` must be shown: that proves `Banner` is in effect.
Only then is it safe to close the session that was kept open.

```
ssh -L 9999:localhost:8080 <ALIAS>
```
Logs in and asks ssh to forward the laptop's port `9999` to port `8080` on the VPS. The
login itself succeeds. Then, from another terminal on the laptop, use the forwarded
port, for example with `curl http://localhost:9999`: the ssh session prints
`channel ... open failed: administratively prohibited`, which proves that
`DisableForwarding yes` is in effect. If it prints
`open failed: connect failed: Connection refused` instead, forwarding still works and
simply nothing is listening on that port.

## 4. Change the SSH port (systemd socket method)
Recent Ubuntu manages the SSH port through `ssh.socket`, not `sshd_config`:
```
sudo systemctl edit ssh.socket
```
> On some terminals (e.g. kitty), prefix with `TERM=xterm-256color` so the editor can
> initialize on the remote host.

Override content (`/etc/systemd/system/ssh.socket.d/override.conf`):
```
[Socket]
ListenStream=
ListenStream=0.0.0.0:<SSH_PORT>
ListenStream=[::]:<SSH_PORT>
```
> Important: declare both IPv4 (`0.0.0.0`) and IPv6 (`[::]`) explicitly. With just
> `ListenStream=<SSH_PORT>` it listened only on IPv6 and IPv4 connections were
> refused.

Apply and verify:
```
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo ss -tlnp | grep <SSH_PORT>
# Two lines expected: 0.0.0.0:<SSH_PORT> and [::]:<SSH_PORT>
```

## 5. Fail2ban
With password login disabled, brute-force attempts cannot succeed, but Fail2ban stays
useful on the VPS: it is reachable from the whole internet, and banning repeat offenders
cuts log noise and load. OpenSSH 9.8+ (e.g. Ubuntu 26.04 LTS) already throttles failing
sources with the built-in `PerSourcePenalties`; on older releases such as Ubuntu 24.04
(OpenSSH 9.6) Fail2ban is the only throttling.

Install:
```
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Config (`/etc/fail2ban/jail.local`):
```
[sshd]
enabled = true
port    = <SSH_PORT>
filter  = sshd
maxretry = 3
findtime = 5m
bantime  = 30m
backend = systemd
```
> `backend = systemd` is key on recent Ubuntu (logs go to the journal, not to
> `/var/log/auth.log`).

Apply and verify:
```
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
sudo fail2ban-client status sshd
```

## 6. Non-root user
The cloud image default admin user already has sudo. No extra user created.

### Note on `sudo` without a password
Cloud images ship a `NOPASSWD` sudo rule for their default user (typically in
`/etc/sudoers.d/90-cloud-init-users`). The reason is that these images have no user
password at all — access is key-only — so a password prompt would make `sudo` unusable.

This was deliberately left as is. The trade-off:

- **In favour of keeping it**: the real barrier is the SSH key protected by a
  passphrase, combined with `PermitRootLogin no` and password authentication disabled.
  Someone without the private key cannot reach a shell in the first place.
- **Against**: any process running as that user can escalate to root with no further
  check. Requiring a password would add a layer of defence in depth.

For a personal VPS reached only through a passphrase-protected key, the convenience is
a reasonable choice. To harden it further, set a real password for the user and remove
the `NOPASSWD` rule (always editing sudoers with `visudo`).

### Note on remote editors and `TERM`
Terminal emulators with non-standard `TERM` values (for example kitty, which sets
`TERM=xterm-kitty`) break ncurses programs on remote hosts that lack the matching
terminfo entry, with errors such as `Error opening terminal: xterm-kitty`.

One-off workaround:
```
sudo TERM=xterm-256color nano <file>
```

Permanent fix, run from the local machine — note it must be installed system-wide
(with `sudo`) so that root also finds it:
```
infocmp -x xterm-kitty | ssh <ALIAS> 'sudo tic -x -'
```
`tic` prints a harmless warning about the description field.

## 7. Firewall (iptables, IPv4 and IPv6)
The model is **deny by default, allow by exception**:

- `INPUT` (traffic addressed to the VPS) and `FORWARD` (traffic passing through it)
  have policy `DROP`: anything not explicitly allowed is discarded.
- `OUTPUT` (traffic the VPS sends) stays `ACCEPT`.

These rules live **outside** WireGuard's `wg0.conf` and are persisted with
`iptables-persistent`, so SSH access never depends on the tunnel being up. `FORWARD`
gets its WireGuard-specific allow rules later, from the `PostUp` hooks in `wg0.conf`
(see [VPS-WIREGUARD.md §5](VPS-WIREGUARD.md#5-configuration-etcwireguardwg0conf)).

### 7.1 Inventory: what is listening
```
sudo ss -tulnp
```
Lists every open socket, to know which services need a rule before closing anything.

- `-t` / `-u` — TCP and UDP sockets.
- `-l` — only listening sockets (services waiting for connections).
- `-n` — numeric addresses and ports, no name resolution.
- `-p` — the process that owns each socket (needs `sudo`).

The `Local Address` column tells who can reach each service: `0.0.0.0` or `[::]` means
all interfaces, so the service is exposed to the internet; `127.0.0.x` or `[::1]` means
loopback only, reachable just from the VPS itself.

What was found on this VPS:

- `sshd` on `<SSH_PORT>`, on `0.0.0.0` and `[::]` (started by systemd through
  `ssh.socket`, §4) → exposed, **needs a rule**.
- `systemd-resolved` on `127.0.0.53:53` and `127.0.0.54:53`, `chronyd` on
  `127.0.0.1:323` and `[::1]:323` → loopback only, no rule needed.
- `systemd-networkd` DHCP client on UDP `68` on the public interface (`<PUBLIC_IFACE>`)
  → no rule needed: the replies to its lease renewals match `ESTABLISHED` (below).
- No DHCPv6 listener → the IPv6 configuration comes from Router Advertisements or is
  static; either way ICMPv6 is required (Neighbor Discovery, Path MTU discovery), see
  §7.3.

> **Connection tracking.** The kernel remembers every connection. `ESTABLISHED` matches
> replies to connections the VPS started itself (apt, DNS through `systemd-resolved`,
> NTP through `chrony`, DHCP renewals) and also keeps the current SSH session alive.
> `RELATED` matches ICMP errors tied to an existing connection, such as "fragmentation
> needed". This is why outbound traffic needs no rules of its own.

### 7.2 Safety net
Keep a **second SSH session** open the whole time, then arm a revert timer:
```
sudo systemd-run --on-active=5min --unit=fw-revert sh -c 'iptables -P INPUT ACCEPT; ip6tables -P INPUT ACCEPT'
```
Schedules a command that reopens `INPUT` in 5 minutes, in case a mistake locks SSH out.

- `systemd-run` — runs a command as a transient systemd unit, independent of the SSH
  session, so it still fires if the session dies.
- `--on-active=5min` — creates a timer that fires 5 minutes from now.
- `--unit=fw-revert` — names the units `fw-revert.timer` / `fw-revert.service`, so the
  timer is easy to check and cancel.
- `sh -c '...'` — runs both commands in one shell. It only resets the `INPUT` policies
  to `ACCEPT`; the rules stay, but that is enough to let SSH back in.

```
systemctl list-timers fw-revert.timer
```
Confirms the timer is armed and shows how much time is left.

To re-arm the timer (for example after it fired), run
`sudo systemctl stop fw-revert.timer` first: a timer that already fired stays loaded,
and `systemd-run` with the same name fails.

### 7.3 Allow rules
Rules are checked **top to bottom and the first match wins**, so they are added before
the policy is switched to `DROP`.

```
sudo iptables -S
sudo ip6tables -S
```
Pre-check: each must print only the three `-P ... ACCEPT` policies and no rules. `-A`
appends, so leftover rules would cause duplicates or rules that are never reached.

> Replace the placeholders before running. Pasting `<SSH_PORT>` literally makes bash
> read `<` as input redirection: it fails with `SSH_PORT: No such file or directory`
> and the command does not run.

IPv4:
```
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport <SSH_PORT> -j ACCEPT
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
```
`-A INPUT` appends each rule to the end of the `INPUT` chain, and `-j ACCEPT` lets the
matching packet in.

- `-i lo` — traffic on the loopback interface, used by local programs to reach local
  services (the `systemd-resolved` stub on `127.0.0.53`, chrony's control port).
- `-m conntrack --ctstate ESTABLISHED,RELATED` — replies and related ICMP errors (see
  the box above).
- `-p tcp --dport <SSH_PORT>` — new SSH connections.
- `-p udp --dport 51820` — WireGuard, opened in advance: nothing listens there yet.
- `-p icmp --icmp-type echo-request` — answers `ping`, useful for diagnostics.

IPv6:
```
sudo ip6tables -A INPUT -i lo -j ACCEPT
sudo ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport <SSH_PORT> -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
```
The same rules for IPv6, except the last one, which allows **all** ICMPv6 instead of only
echo requests. IPv6 cannot work without ICMPv6: it carries Neighbor Discovery (IPv6's
equivalent of ARP), Router Advertisements (if the IPv6 setup is not static) and Path MTU
discovery.

```
sudo iptables -S INPUT
sudo ip6tables -S INPUT
```
Prints the `INPUT` chain as rules. Check that all five rules are there, each exactly
once, and that the SSH port matches the one shown by `ss`.

### 7.4 Default policies
```
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo ip6tables -P INPUT DROP
sudo ip6tables -P FORWARD DROP
```
`-P` sets a chain's default policy: what happens to a packet that matched no rule. From
now on, anything not allowed in §7.3 is dropped. The current SSH session survives
because it matches `ESTABLISHED`.

> This cloud image enables IPv4 and IPv6 forwarding at boot on its own (no sysctl file
> sets it). `FORWARD DROP` is what keeps the VPS from routing traffic for others.

### 7.5 Test from a new terminal
```
ssh <ALIAS>
```
Run it from a **new** terminal on the laptop. A new connection proves that the SSH rule
works; an already open session only proves `ESTABLISHED`. If it fails, do not save
anything: wait for the timer to reopen `INPUT`, fix the rules and re-arm the timer
(§7.2) before switching the policies to `DROP` again.

### 7.6 Persist the rules
```
sudo systemctl stop fw-revert.timer
systemctl list-timers fw-revert.timer
```
Cancels the revert timer **first**. The second command must print `0 timers listed`.

```
sudo iptables -S | head -2
sudo ip6tables -S | head -2
```
Shows the first lines of the ruleset, which are the policies. `INPUT` and `FORWARD` must
still be `DROP`.

```
sudo iptables -S | grep f2b
sudo ip6tables -S | grep f2b
```
Both must print nothing. If Fail2ban uses iptables actions, its `f2b-*` chains would
otherwise be saved into `rules.v4`/`rules.v6` and clash with the ones Fail2ban creates
at boot.

```
sudo apt install iptables-persistent
```
Installs `netfilter-persistent`, which reloads `/etc/iptables/rules.v4` and
`/etc/iptables/rules.v6` at boot. During the install, answer **Yes** to saving the
current IPv4 and IPv6 rules. If `ufw` is installed, apt proposes to remove it, because
the two conflict; that is expected (ufw is not used here). If the package is already
installed, save with:
```
sudo netfilter-persistent save
```
Writes the rules currently in the kernel to those two files.

```
sudo grep -E ':INPUT|:FORWARD' /etc/iptables/rules.v4 /etc/iptables/rules.v6
```
Shows the saved policies. In the `filter` section `INPUT` and `FORWARD` must be `DROP`;
the policy shown for the `mangle` table is irrelevant here.

> **Lesson learned: cancel the timer → check the policy → save.** If the timer fires
> before it is cancelled, `INPUT` goes back to `ACCEPT`, and saving afterwards makes the
> open firewall permanent. This happened once during this rebuild and was caught by the
> reboot check below.

> Never run `netfilter-persistent save` while `wg0` is up: the `PostUp` rules would be
> saved too, loaded at boot and then added again by `PostUp`, leaving duplicate rules
> that `PostDown` does not fully remove.

### 7.7 Reboot and verify
```
sudo reboot
```
Only a real reboot proves that the rules are loaded at boot. After reconnecting:
```
sudo iptables -S
sudo ip6tables -S
```
Expected output for IPv4:
```
-P INPUT DROP
-P FORWARD DROP
-P OUTPUT ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport <SSH_PORT> -j ACCEPT
-A INPUT -p udp -m udp --dport 51820 -j ACCEPT
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
```
And for IPv6:
```
-P INPUT DROP
-P FORWARD DROP
-P OUTPUT ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport <SSH_PORT> -j ACCEPT
-A INPUT -p udp -m udp --dport 51820 -j ACCEPT
-A INPUT -p ipv6-icmp -j ACCEPT
```
`iptables -S` prints rules in its own normalised form: it adds the implicit `-m tcp` /
`-m udp` matches, lists the states as `RELATED,ESTABLISHED` and shows `echo-request` as
type `8`.

---

## Related documents
- [VPS-WIREGUARD.md](VPS-WIREGUARD.md) — the WireGuard hub built on top of this firewall.
- [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) — preparing the Raspberry Pi: OS on NVMe and
  SSH hardening (the reasoning behind each sshd directive).
