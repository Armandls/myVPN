# Preparing the Raspberry Pi — OS on NVMe and SSH hardening

How the Raspberry Pi 5 is prepared before any VPN or homelab work: install Raspberry Pi
OS Lite on a microSD card, clone the whole system to an NVMe drive with `rpi-clone`, boot
from the NVMe from then on, and finally harden SSH access to the Pi (key-only login for a
single user, no root login).

The SSH hardening of the VPS is documented separately in [HARDENING.md](HARDENING.md);
this document covers its Pi counterpart.

This document follows the same convention as [SETUP.md](SETUP.md): every command is
listed with an explanation of what it does and why. Placeholders (`<USER>`,
`<PI_HOSTNAME>`, ...) are defined in the
[SETUP.md placeholders table](SETUP.md#placeholders-used).

When this is done, continue with
[SETUP.md Phase 2](SETUP.md#phase-2--raspberry-pi-reverse-tunnel--lan-gateway) (WireGuard on
the Pi) and then [PIHOLE.md](PIHOLE.md) (Docker + Pi-hole + Unbound).

---

## Key concepts

A few terms that appear throughout this document:

- **EEPROM bootloader** — the Pi 5 does not boot straight from a disk. A small program
  stored in a chip on the board (the EEPROM) runs first, decides which device to boot
  from, and loads the firmware and kernel from it. Because it lives on the board and not
  on any disk, its settings (such as the boot order) survive reflashing or swapping
  disks. It is updated with `rpi-eeprom-update` and configured with `rpi-eeprom-config`.
- **Boot partition and root partition** — Raspberry Pi OS uses two partitions: a small
  FAT partition mounted at `/boot/firmware` (firmware, kernel, `config.txt`,
  `cmdline.txt`) and a large ext4 partition mounted at `/` (the rest of the system).
- **PARTUUID** — an identifier written into the partition table for each partition. The
  system finds its partitions by PARTUUID rather than by device name: `cmdline.txt`
  tells the kernel `root=PARTUUID=...`, and `/etc/fstab` mounts `/boot/firmware` the same
  way. Device names such as `mmcblk0` or `nvme0n1` can change between boots; a PARTUUID
  does not. The catch: if two disks carry the same PARTUUIDs, the system cannot tell them
  apart.

---

## Design decisions

### Everything on NVMe, booting directly from it

Both `/` and `/boot/firmware` live on the NVMe drive, and the Pi boots from it with no
microSD involved.

- The Pi 5 bootloader supports NVMe boot natively. Keeping `/boot` on the SD card and `/`
  on another disk was a workaround from the Pi 3/4 era; on a Pi 5 it only adds a second
  point of failure.
- microSD cards wear out under constant writes (logs, journald, Docker, Pi-hole) and
  tend to corrupt on power loss. An NVMe drive is far more durable and faster.
- A single disk keeps things simple for Docker: its data stays in the default
  `/var/lib/docker`, so there is no need to relocate containerd's store or to order
  services after a separate mount.

### Install on microSD first, then clone

Instead of flashing the NVMe directly, the OS is installed on a microSD card and then
copied. The Raspberry Pi Imager preconfiguration (user, SSH, hostname) is done once and
travels with the copy, and the microSD remains afterwards as a working rescue disk.

### Clone right after first boot

Copying a *running* system is never perfectly clean: while the copy is in progress,
services keep writing files (logs, caches, databases), so some files may be copied
half-written. A freshly installed, nearly idle system writes very little, which makes
this the safest moment to clone — before installing anything else.

### `rpi-clone` rather than `dd`

`dd` copies a disk byte for byte, which causes two problems here:

1. **Size.** The root partition keeps the size it had on the microSD. It has to be grown
   by hand afterwards (`growpart` + `resize2fs`) to use the rest of the NVMe.
2. **Duplicate PARTUUIDs.** Both disks end up with identical PARTUUIDs. Since
   `cmdline.txt` and `/etc/fstab` locate partitions by PARTUUID, booting with the SD card
   still inserted can load the firmware from the NVMe but mount `/` from the SD — without
   any visible error.

`rpi-clone` avoids both: it creates the partitions on the destination, copies the files
with `rsync`, grows the last partition to fill the disk, and gives the clone new
PARTUUIDs, updating `cmdline.txt` and `/etc/fstab` on the destination to match.

### A maintained, Pi 5–aware fork of `rpi-clone`

The original `billw2/rpi-clone` has been unmaintained since 2020. It predates the
`/boot/firmware` layout (it expects the boot partition at `/boot`), which was introduced
with Bookworm and is kept in the current Raspberry Pi OS, based on Debian 13 "trixie" —
the version installed here. It also has no clear NVMe support. The maintained fork
[geerlingguy/rpi-clone](https://github.com/geerlingguy/rpi-clone) handles both; version
2.0.27 was used here.

### SSH settings in our own drop-in file, numbered `40-`

The SSH settings are not edited in the main `/etc/ssh/sshd_config` nor in the
`50-cloud-init.conf` file that the Imager preconfiguration creates. They go in a separate
file, `/etc/ssh/sshd_config.d/40-myvpn.conf`:

- **sshd keeps the first value it finds.** For most settings, sshd uses the *first* value
  it reads and ignores any later one. The exception is the `Allow*`/`Deny*` lists
  (`AllowUsers`, `DenyUsers`, `AllowGroups`, `DenyGroups`), which are accumulated across
  files: another drop-in adding its own `AllowUsers` would widen access. The main `sshd_config` includes the files in
  `sshd_config.d/` at the top, before its own settings, and those files are read in
  lexical (alphabetical) order. So a file named `40-...` is read before `50-...`, and its
  values win.
- **cloud-init sets `PasswordAuthentication yes`.** The Imager settings are applied on
  first boot by cloud-init, which writes `50-cloud-init.conf` with password login
  enabled. A file numbered `99-` would be read after it and lose; numbered `40-`, it wins
  whatever cloud-init writes. On the VPS, [HARDENING.md](HARDENING.md) uses
  `99-hardening.conf`, which works there only because the competing `50-`/`60-` files
  were removed and cloud-init was neutralised.
- **Files managed by cloud-init can be rewritten.** Editing `50-cloud-init.conf` is
  fragile, because cloud-init may regenerate it. With the `40-` file in place,
  `50-cloud-init.conf` can stay as it is.

---

## Step 1 — Flash Raspberry Pi OS Lite to the microSD

On the laptop, open **Raspberry Pi Imager** and choose:

- Device: Raspberry Pi 5.
- OS: **Raspberry Pi OS Lite (64-bit)** — no desktop, since the Pi runs headless.
- Storage: the microSD card.

In the OS customisation settings, preconfigure the user `<USER>`, the hostname
`<PI_HOSTNAME>` and enable **SSH** with password authentication; it is hardened to key-only login in
[Step 11](#step-11--harden-ssh). This lets the Pi
be reached over the network on first boot without a screen or keyboard, and these
settings are carried over to the NVMe by the clone.

## Step 2 — First boot and update

Insert the microSD, power on the Pi and connect from the laptop:
```bash
ssh <USER>@<PI_HOSTNAME>.local
```
Opens a shell on the Pi with the user created by the Imager. The `.local` suffix is the
mDNS name the Pi announces on the LAN; the plain hostname only resolves if the router
registers DHCP client names. If neither works, look up the Pi's address (`<PI_LAN_IP>`)
in the router's DHCP client list and use `ssh <USER>@<PI_LAN_IP>`.

```bash
sudo apt update && sudo apt full-upgrade
```
Refreshes the package lists and upgrades every installed package, including kernel and
firmware. The system is about to be copied, so it should be copied up to date.

```bash
sudo rpi-eeprom-update
```
Reports the installed EEPROM bootloader version and the latest available one. A recent
bootloader has the best NVMe boot support. If it reports an update as pending,
`sudo rpi-eeprom-update -a` applies the update, effective from the next reboot.

```bash
sudo reboot
```
Restarts the Pi so the new kernel, firmware and (if updated) bootloader are actually
running before the clone. If something breaks after the upgrade, it shows up now, on the
SD card, instead of being mixed up with a possible clone problem later.

## Step 3 — Identify the disks

```bash
lsblk
```
Lists the block devices and where their partitions are mounted. Here:

| Device | What it is |
|---|---|
| `mmcblk0` | the microSD — the booted disk |
| `mmcblk0p1` | `/boot/firmware` (512M) |
| `mmcblk0p2` | `/` |
| `nvme0n1` | the NVMe drive — the clone destination |

**This is the critical step.** The destination disk is wiped completely; choosing the
wrong one destroys the wrong data. Confirm which device is mounted at `/` (the source)
and which one is the NVMe before going further.

The NVMe here still had an old partition (`nvme0n1p1`, old Docker data). It does not need
to be wiped by hand: it is not mounted, so it does no harm, and the `-f` option in step 5
re-creates the partition table and overwrites it.

## Step 4 — Install `rpi-clone` (maintained fork)

The install method is described in the fork's
[README](https://github.com/geerlingguy/rpi-clone). The manual install from source was
used here:
```bash
git clone https://github.com/geerlingguy/rpi-clone.git ~/rpi-clone
cd ~/rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin
```
Downloads the fork and copies its two scripts into `/usr/local/sbin`, which is on
`root`'s `PATH`. The README also offers a one-line
`curl ... | sudo bash` installer; the manual method lets the script be read before it is
run as `root`.

```bash
sudo rpi-clone -V
git -C ~/rpi-clone remote -v
```
The first prints the installed version (2.0.27 here); it runs with `sudo` because
`/usr/local/sbin` may not be on a normal user's `PATH`, and the clone itself runs as
`root` anyway. The second confirms the cloned repository came from
`geerlingguy/rpi-clone` and not from the unmaintained original.

## Step 5 — Clone the microSD to the NVMe

```bash
sudo rpi-clone nvme0n1 -f
```
Clones the booted disk (the microSD) to `nvme0n1`.

- `nvme0n1` is the **whole disk**, not a partition (`nvme0n1p1`). `rpi-clone` creates the
  partitions itself.
- `-f` forces an *initialisation* clone: the destination partition table is rebuilt by
  imaging the booted disk's partition structure, and fresh filesystems are created
  before syncing. This is what discards the old partition on the NVMe.
- `-f2` is not needed: it is meant for cloning a disk with many partitions onto a
  two-partition layout, and the microSD already has exactly two.
- **No `-U` the first time.** Without it, `rpi-clone` shows its plan and waits for
  confirmation. That confirmation is the safety check; skipping it means trusting the
  device name blindly.

Before copying anything it prints a pre-flight table. What was shown here, and how to
read it:

- **Booted disk `mmcblk0` → destination `nvme0n1`.** The source must be the microSD and
  the destination the NVMe. If either is different, answer `no`.
- **`Initialize: IMAGE partition table - forced by option`** — the partition table will
  be recreated because of `-f`.
- **Partition 1 `/boot/firmware` : `MKFS SYNC to nvme0n1p1`** — a new FAT filesystem is
  created and the boot files are copied into it.
- **Partition 2 `root` : `RESIZE MKFS SYNC to nvme0n1p2`** — the root partition is grown
  to fill the NVMe, a new ext4 filesystem is created and `/` is copied into it.
- **`WARNING`: all destination data will be overwritten.**

Only after checking source and destination, answer `yes`.

## Step 6 — Filesystem label

`rpi-clone` then asks for an optional label for the ext filesystem. Press **Enter** to
leave it blank: the system identifies partitions by PARTUUID, so a label is not needed.

## Step 7 — Check the clone before unmounting

When the copy finishes, `rpi-clone` leaves the destination mounted (under `/mnt/clone`
by default) and waits with:

```
Hit Enter when ready to unmount the /dev/nvme0n1 partitions
```

Before pressing Enter, open **another SSH session** and inspect the clone:
```bash
sudo blkid /dev/nvme0n1p1 /dev/nvme0n1p2
```
Prints the identifiers of the two new NVMe partitions, including their `PARTUUID`.

```bash
cat /mnt/clone/boot/firmware/cmdline.txt
```
Shows the kernel command line the NVMe will boot with. Its `root=PARTUUID=...` must
match the `PARTUUID` of `nvme0n1p2` from `blkid`.

```bash
cat /mnt/clone/etc/fstab
```
Shows how the cloned system will mount its partitions. The PARTUUIDs for `/` and
`/boot/firmware` must match `nvme0n1p2` and `nvme0n1p1`.

In both files the PARTUUIDs must belong to the NVMe and be **different** from the
microSD's (`sudo blkid /dev/mmcblk0p1 /dev/mmcblk0p2` shows those). That proves the
clone will mount its own partitions and cannot be confused with the SD card. Then go back
to the first session and press Enter.

## Step 8 — Put NVMe first in the boot order

```bash
sudo rpi-eeprom-config --edit
```
Opens the bootloader configuration in an editor. Set:
```
BOOT_ORDER=0xf416
```
Saving applies the new configuration; it takes effect on the next boot.

`BOOT_ORDER` is a hexadecimal number read **from right to left**, one digit per boot
attempt:

| Digit | Meaning |
|---|---|
| `6` | NVMe — tried first |
| `1` | SD card — tried second |
| `4` | USB — tried third |
| `f` | restart the loop from the right |

Without this change, and with the microSD still inserted, the Pi will keep booting from
the SD card and nothing would look wrong: the default order is `0xf461`, which tries the
SD card (`1`) before NVMe (`6`). With NVMe first, the SD only acts as a
fallback.

## Step 9 — Boot from NVMe

```bash
sudo poweroff
```
Shuts the Pi down cleanly. Then **remove the microSD** and power the Pi on again. With no
SD card inserted, a successful boot can only have come from the NVMe.

## Step 10 — Verify

```bash
findmnt /
findmnt /boot/firmware
```
Show which device each mount point comes from: `/` must be `/dev/nvme0n1p2` and
`/boot/firmware` must be `/dev/nvme0n1p1`.

```bash
df -h /
```
Shows the size of the root filesystem. It must be roughly the whole NVMe (about `117G` on
a 128 GB drive), proving `rpi-clone` grew the partition instead of keeping the microSD
size.

```bash
rpi-eeprom-config | grep BOOT_ORDER
```
Prints the boot order stored in the bootloader; it must show `BOOT_ORDER=0xf416`. This
check is needed because, with the microSD removed, the Pi boots from NVMe even with the
default order, so `findmnt` alone does not prove that step 8 was applied.

```bash
sudo rpi-eeprom-update
```
Shows the bootloader version currently running and whether a newer one is available,
confirming the update from step 2 is in place.

Keep the microSD aside: it holds a bootable copy of the freshly installed system and can
be used as a rescue disk. It was cloned before the SSH hardening of step 11, so it still
accepts password logins. That only matters to someone with physical access to the Pi,
but if it is kept, harden it the same way or wipe it.

## Step 11 — Harden SSH

Until now the Pi accepts SSH logins with a password. This step switches it to key-only
login, for `<USER>` only, with root login and SSH forwarding disabled. The order
matters: the key is installed and tested **before** passwords are turned off, so access
is never lost.

### 11.1 Generate a key pair on the laptop

```bash
ssh-keygen -t ed25519 -C "<comment>" -f ~/.ssh/raspberry
```
Creates a new key pair dedicated to the Pi, on the laptop.

- `-t ed25519` selects the Ed25519 algorithm: modern, secure and with short keys.
- `-C "<comment>"` is only a label stored in the public key, to recognise it later (for
  example in `authorized_keys`). It has no security role.
- `-f ~/.ssh/raspberry` chooses the file name: the private key is `~/.ssh/raspberry` and
  the public key `~/.ssh/raspberry.pub`.

`ssh-keygen` then asks for a **passphrase** at an interactive prompt. The passphrase
encrypts the private key on disk, so a stolen copy of the file is useless without it.
Type it at the prompt rather than passing it with `-N "..."`: anything written on the
command line is saved in the shell history.

### 11.2 Copy the public key to the Pi

```bash
ssh-copy-id -i ~/.ssh/raspberry.pub <USER>@<PI_LAN_IP>
```
Appends the public key to `~/.ssh/authorized_keys` of `<USER>` on the Pi, creating the
file and directory with the correct permissions if needed. It logs in with the password
to do so, so it must run while password login is still allowed.

### 11.3 Test key login

```bash
ssh -i ~/.ssh/raspberry -o IdentitiesOnly=yes -o PasswordAuthentication=no <USER>@<PI_LAN_IP>
```
Logs in with the new key (asking for its passphrase, not the account password).
`IdentitiesOnly=yes` makes ssh offer only that key, not keys from an agent or the default
`~/.ssh/id_*` files, and `PasswordAuthentication=no` stops it falling back to a password.
So a successful login proves that this specific key works, and passwords can be disabled
safely. Keep this session open
for the next steps.

### 11.4 Create the drop-in file

On the Pi, create the file:
```bash
sudo nano /etc/ssh/sshd_config.d/40-myvpn.conf
```
Opens a new, empty file in the `nano` editor as `root` (the directory is only writable by
`root`). Write these contents and save:
```
# Only <USER> may log in over SSH
AllowUsers <USER>

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

- `AllowUsers <USER>` — only this account may log in over SSH; any other user is
  rejected even with valid credentials.
- `PubkeyAuthentication yes` — allows login with a key pair.
- `PasswordAuthentication no` — rejects password logins, which removes password
  guessing as an attack.
- `AuthenticationMethods publickey` — every login must succeed with a key. This is a
  second, independent lock besides `PasswordAuthentication no`: even if another file
  re-enabled a password method, it would not be accepted, unless that file also
  overrides `AuthenticationMethods`.
- `PermitRootLogin no` — `root` cannot log in over SSH; administration goes through
  `<USER>` and `sudo`.
- `KbdInteractiveAuthentication no` — keyboard-interactive is the other way SSH can ask
  for a password (through PAM). Debian's main `sshd_config` already sets it to `no`, but
  that line comes **after** the `Include` of `sshd_config.d/`, so another drop-in could
  override it. Setting it here pins it.
- `DisableForwarding yes` — turns off every forwarding feature of sshd: TCP port
  forwarding (`ssh -L`, `-R`, `-D`), ssh-agent forwarding, X11 and Unix-socket
  (StreamLocal) forwarding. It overrides `AllowTcpForwarding`, `AllowAgentForwarding`,
  `X11Forwarding` and `AllowStreamLocalForwarding`. It does not cover `PermitTunnel`
  (tun devices, `ssh -w`), which is already `no` by default. It has nothing to do with
  the kernel's `net.ipv4.ip_forward`: the Pi's role as LAN gateway and WireGuard
  routing are not affected. The trade-off is that the Pi cannot be used as a `ProxyJump`
  host, and tools that rely on SSH tunnels (`ssh -L`, VS Code Remote-SSH) do not work.
  They are not needed here, because the VPN already gives direct access to the LAN and
  to Pi-hole's `:8080`. If forwarding is ever needed, it can be re-enabled narrowly
  inside a `Match User`/`Match Address` block.
- `AuthorizedKeysFile .ssh/authorized_keys` — sshd only reads keys from
  `~/.ssh/authorized_keys`. Debian's default also reads `~/.ssh/authorized_keys2`, a
  legacy file nobody checks, where a key could be planted unnoticed. Limiting it to one
  file leaves a single place to audit.
- `Banner /etc/ssh/banner` — sshd sends the text in this file to anyone who connects,
  **before** authentication. It is typically an authorised-use notice, and it must not
  reveal anything useful to an attacker (hostname, OS, hardware, versions). It lives in
  `/etc/ssh/`, owned by `root`, and not in the user's home: sshd reads it as `root`, in
  its privileged process, before authentication, and follows symlinks. A file in the
  user's home could be replaced by a symlink to a root-only file (for example
  `/etc/shadow` or a WireGuard private key) by anything running as the user, and sshd
  would then show that file's content to anyone who connects.

Why the file is numbered `40-` is explained in
[Design decisions](#ssh-settings-in-our-own-drop-in-file-numbered-40-).

Create the banner referenced by `Banner`:
```bash
sudo nano /etc/ssh/banner
```
Opens a new file as `root`, so it is created owned by `root`. Write a short
authorised-use notice and save, for example:
```
Authorized access only. All connections may be monitored and logged.
```

```bash
sudo chown root:root /etc/ssh/banner
sudo chmod 644 /etc/ssh/banner
```
Makes the banner owned by `root`, world-readable like the other files in `/etc/ssh/`,
and writable only by `root`. The `chown` matters if the file was created or moved there
by the normal user: otherwise that user could still replace its content.

```bash
sudo chmod 600 /etc/ssh/sshd_config.d/40-myvpn.conf
```
Makes the file readable (and writable) only by `root`. The default `644` already makes it
editable only by `root`; `600` also stops other local users from reading it, for example
to learn which account is in `AllowUsers`. It is not a secret, so this is not strictly
required; it is kept consistent with `50-cloud-init.conf`. As a consequence, `sshd -T`
(11.6) must be run with `sudo`: without it, sshd cannot read the file and prints
`/etc/ssh/sshd_config.d/40-myvpn.conf: Permission denied`.

### 11.5 Check the syntax, then restart

```bash
sudo sshd -t
```
Checks the whole SSH configuration for errors without applying it. It prints nothing
when everything is correct. Always run it **before** restarting, so sshd is never
restarted with a broken configuration.

```bash
sudo systemctl restart ssh
```
Restarts the SSH server so it reads the new configuration. On Debian and Raspberry Pi
OS the unit is called `ssh` (`sshd` is an alias). Restarting does not drop existing
sessions: keep the current one open until the tests below pass, since it is the way back
if something is wrong.

### 11.6 Check the effective configuration

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication|permitrootlogin|allowusers|kbdinteractive|authenticationmethods|disableforwarding|authorizedkeysfile|banner'
```
`-T` prints the final configuration sshd actually uses, after merging every file (unlike
`-t`, which only checks syntax). It needs `sudo` because `40-myvpn.conf` is readable only
by `root`. Expected output (the order of the lines may differ):
```
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
disableforwarding yes
banner /etc/ssh/banner
authorizedkeysfile .ssh/authorized_keys
allowusers <USER>
authenticationmethods publickey
```
`kbdinteractiveauthentication no` is pinned on purpose (see 11.4); the VPS does the same
in [HARDENING.md](HARDENING.md).

The full `sshd -T` output still lists `x11forwarding`, `allowtcpforwarding`,
`allowagentforwarding` and `allowstreamlocalforwarding` as `yes`. This is expected:
`disableforwarding yes` overrides them, so they have no effect.

### 11.7 Test from the laptop

From a **new** terminal on the laptop:
```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive <USER>@<PI_LAN_IP>
```
Tries to log in using both password methods, ignoring keys. It must fail with
`Permission denied (publickey)`. That message is the real proof: the text in parentheses
is the list of methods the server offers, and here it offers only `publickey`.

```bash
ssh -i ~/.ssh/raspberry -o IdentitiesOnly=yes -o PasswordAuthentication=no <USER>@<PI_LAN_IP>
```
Logs in with that key only, with no password fallback, as in 11.3 (asking for the key's
passphrase). It must succeed. Before the passphrase prompt, the text of
`/etc/ssh/banner` must be shown: that proves `Banner` is in effect. Only then is it safe
to close the session that was kept open.

```bash
ssh -i ~/.ssh/raspberry -o IdentitiesOnly=yes -L 9999:localhost:8080 <USER>@<PI_LAN_IP>
```
Logs in with the key and asks ssh to forward the laptop's port `9999` to port `8080` on
the Pi. The login itself succeeds. Then, from another terminal on the laptop, use the
forwarded port, for example with `curl http://localhost:9999`: the connection fails and
the ssh session prints `channel ... open failed: administratively prohibited`. This
proves that `DisableForwarding yes` is in effect. If ssh instead prints
`open failed: connect failed: Connection refused`, forwarding is still enabled and simply
nothing is listening on port `8080` yet (Pi-hole is installed later), so only
`administratively prohibited` proves that `DisableForwarding` works.

---

## Next steps

- [SETUP.md Phase 2](SETUP.md#phase-2--raspberry-pi-reverse-tunnel--lan-gateway) — WireGuard on the Pi (reverse tunnel + LAN gateway).
- [PIHOLE.md](PIHOLE.md) — Docker + Pi-hole + Unbound.

## Related documents

- [SETUP.md](SETUP.md) — the VPN itself, command by command.
- [PLAN.md](PLAN.md) — phases and design reasoning.
- [OPERATIONS.md](OPERATIONS.md) — day-to-day usage.
- [HARDENING.md](HARDENING.md) — SSH hardening of the VPS (the counterpart of step 11).
