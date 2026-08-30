# VPS SSH Hardening

Record of the securization steps applied to the VPS before setting up WireGuard.

## Server details
- **Provider**: cloud VPS (2 vCore, 4 GB RAM, 40 GB NVMe, unlimited traffic).
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
ssh-keygen -t ed25519 -a 100 -C "vps-key" -f ~/.ssh/<KEY_NAME>
```
- `~/.ssh/<KEY_NAME>` → private key (permissions 600).
- `~/.ssh/<KEY_NAME>.pub` → public key (permissions 644).

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

Custom hardening file:
```
# file: /etc/ssh/sshd_config.d/99-hardening.conf
PasswordAuthentication no  # Don't accept a normal account password over SSH
PermitRootLogin no  # Don't allow the root account to log in via SSH
PubkeyAuthentication yes  # Allow SSH key authentication
KbdInteractiveAuthentication no  # Don't allow keyboard-interactive authentication
```

Apply and verify:
```
sudo sshd -t
sudo systemctl restart ssh
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
# Expected:
#   permitrootlogin no
#   passwordauthentication no
```

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
infocmp -x xterm-kitty | ssh <HOST> 'sudo tic -x -'
```
`tic` prints a harmless warning about the description field. See
[TROUBLESHOOTING.md](TROUBLESHOOTING.md) issue 4.

---

## Related documents
- [SETUP.md](SETUP.md) — WireGuard setup commands.
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — the SSH issues hit here are documented as
  issues 1 to 5.
- [OPERATIONS.md](OPERATIONS.md) — ongoing maintenance.
