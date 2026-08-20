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
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
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
Note: cloud images ship `NOPASSWD` sudo for that user; kept as is since the strong
barrier is the SSH key + passphrase and `PermitRootLogin no`.

---

## Firewall (iptables) — Level B: default DROP
Base security rules persisted with `iptables-persistent` (independent of WireGuard):
```
sudo apt install iptables-persistent -y

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport <SSH_PORT> -j ACCEPT     # SSH
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT          # WireGuard
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT   # ping

# Only after verifying SSH still works in another terminal:
sudo iptables -P INPUT DROP

sudo netfilter-persistent save
```
> Golden rule: add all ACCEPT rules and verify SSH BEFORE switching the policy to
> DROP, otherwise you lock yourself out. Keep one SSH session open as a safety net.

FORWARD + MASQUERADE rules live in `wg0.conf` (`PostUp`/`PostDown`), managed by
`wg-quick`.

---

## Hardening status
- [x] Step A — SSH password/root login disabled.
- [x] Step B — SSH port changed (IPv4+IPv6).
- [x] Step C — fail2ban active and verified.
- [x] Step D — non-root sudo user.
- [x] Firewall Level B — default DROP + rules, persisted and verified across reboot.
