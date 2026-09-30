# Preparing the Raspberry Pi — OS on NVMe

How the Raspberry Pi 5 gets its operating system before any VPN or homelab work: install
Raspberry Pi OS Lite on a microSD card, clone the whole system to an NVMe drive with
`rpi-clone`, and boot from the NVMe from then on.

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

### A Bookworm / Pi 5–aware fork of `rpi-clone`

The original `billw2/rpi-clone` has been unmaintained since 2020. It predates Bookworm
(it expects the boot partition at `/boot`, not `/boot/firmware`) and has no clear NVMe
support. The maintained fork
[geerlingguy/rpi-clone](https://github.com/geerlingguy/rpi-clone) handles both; version
2.0.27 was used here.

---

## Step 1 — Flash Raspberry Pi OS Lite to the microSD

On the laptop, open **Raspberry Pi Imager** and choose:

- Device: Raspberry Pi 5.
- OS: **Raspberry Pi OS Lite (64-bit)** — no desktop, since the Pi runs headless.
- Storage: the microSD card.

In the OS customisation settings, preconfigure the user `<USER>`, the hostname
`<PI_HOSTNAME>` and enable **SSH** (public-key authentication preferred). This lets the Pi
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
be used as a rescue disk.

---

## Next steps

- [SETUP.md Phase 2](SETUP.md#phase-2--raspberry-pi-reverse-tunnel--lan-gateway) — WireGuard on the Pi (reverse tunnel + LAN gateway).
- [PIHOLE.md](PIHOLE.md) — Docker + Pi-hole + Unbound.

## Related documents

- [SETUP.md](SETUP.md) — the VPN itself, command by command.
- [PLAN.md](PLAN.md) — phases and design reasoning.
- [OPERATIONS.md](OPERATIONS.md) — day-to-day usage.
