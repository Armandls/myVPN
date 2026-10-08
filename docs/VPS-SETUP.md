# VPS setup — SSH hardening

Record of the hardening steps applied to the VPS before setting up WireGuard.

> The VPS firewall and WireGuard are being rebuilt and will be documented separately.

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

---

## Related documents
- [RASPBERRY-SETUP.md](RASPBERRY-SETUP.md) — preparing the Raspberry Pi: OS on NVMe and
  SSH hardening (the reasoning behind each sshd directive).
