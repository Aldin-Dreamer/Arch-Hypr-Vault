# Arch Linux — Encrypted Install with LUKS2 + TPM2, Secure Boot, UKI & Btrfs

A complete, reproducible Arch Linux installation guide featuring full-disk encryption
unlocked by TPM2, Secure Boot with your own keys, Unified Kernel Images, and a Btrfs
subvolume layout with snapshot support.

This guide documents my personal setup. Scripts in `scripts/` automate most of the
process, but the README walks through every step manually so you understand what each
script is actually doing before you run it.

> **⚠️ Warning:** This process will completely wipe the target disk.
> Read through the entire guide at least once before you begin.
> Do not run scripts on the wrong disk.

---

## Table of Contents

1. [What This Setup Achieves](#1-what-this-setup-achieves)
2. [How It All Fits Together](#2-how-it-all-fits-together)
3. [Prerequisites](#3-prerequisites)
4. [Repo Structure](#4-repo-structure)
5. [Pre-Installation](#5-pre-installation)
   - 5.1 [Boot the Arch ISO](#51-boot-the-arch-iso)
   - 5.2 [Verify UEFI Mode](#52-verify-uefi-mode)
   - 5.3 [Connect to the Internet](#53-connect-to-the-internet)
   - 5.4 [Update the System Clock](#54-update-the-system-clock)
6. [Disk Partitioning](#6-disk-partitioning)
7. [LUKS2 Encryption](#7-luks2-encryption)
8. [Btrfs Setup](#8-btrfs-setup)
   - 8.1 [Format and Mount](#81-format-and-mount)
   - 8.2 [Subvolume Layout](#82-subvolume-layout)
   - 8.3 [Mount Options](#83-mount-options)
9. [Base System Installation](#9-base-system-installation)
10. [System Configuration (chroot)](#10-system-configuration-chroot)
11. [Bootloader — systemd-boot](#11-bootloader--systemd-boot)
12. [Unified Kernel Image (UKI)](#12-unified-kernel-image-uki)
    - 12.1 [What is a UKI and Why Use One](#121-what-is-a-uki-and-why-use-one)
    - 12.2 [Configuring mkinitcpio](#122-configuring-mkinitcpio)
    - 12.3 [Building the UKI](#123-building-the-uki)
13. [Secure Boot](#13-secure-boot)
    - 13.1 [Enrolling Your Own Keys with sbctl](#131-enrolling-your-own-keys-with-sbctl)
    - 13.2 [Signing the UKI](#132-signing-the-uki)
    - 13.3 [Automating Re-signing on Kernel Updates](#133-automating-re-signing-on-kernel-updates)
14. [TPM2 Enrollment](#14-tpm2-enrollment)
    - 14.1 [Understanding PCR Policies](#141-understanding-pcr-policies)
    - 14.2 [Enrolling the LUKS Volume](#142-enrolling-the-luks-volume)
    - 14.3 [Testing and Fallback](#143-testing-and-fallback)
15. [Snapper — Btrfs Snapshots](#15-snapper--btrfs-snapshots)
16. [Post-Installation Checklist](#16-post-installation-checklist)
17. [Recovery Guide](#17-recovery-guide)
18. [Troubleshooting](#18-troubleshooting)

---

## 1. What This Setup Achieves

| Feature | Implementation |
|---|---|
| Full-disk encryption | LUKS2 on the root partition |
| Automatic unlock at boot | TPM2 via `systemd-cryptenroll` bound to PCR 7+11 |
| Fallback unlock | LUKS passphrase |
| Tamper-evident boot | Secure Boot with personal PK/KEK/db keys |
| Unified boot image | UKI — kernel + initramfs + cmdline in one signed `.efi` |
| Bootloader | `systemd-boot` |
| Filesystem | Btrfs with subvolumes and `zstd` compression |
| Snapshot support | Snapper with automatic pre/post and timeline snapshots |

---

## 2. How It All Fits Together

Understanding the boot chain before you start will save you a lot of confusion:

```
Power on
  └─► UEFI firmware
        └─► Checks Secure Boot signatures
              └─► Loads systemd-boot (signed .efi)
                    └─► Loads your UKI (signed .efi)
                          └─► Kernel + initramfs boot
                                └─► systemd-cryptsetup asks TPM2 to unseal the LUKS key
                                      ├─► TPM2 checks PCR values (firmware + boot config unchanged?)
                                      │     ├─► Yes → unlocks automatically
                                      │     └─► No  → falls back to passphrase prompt
                                      └─► Btrfs root mounted → userspace starts
```

The critical insight: **TPM2 will only release the key if the PCR values match what was
recorded at enrollment time.** If you update your kernel (changing the UKI) or touch
Secure Boot keys, you must re-enroll the TPM2 binding. The scripts handle this.

→ See [docs/boot-chain-explained.md](docs/boot-chain-explained.md) for a deeper breakdown.

---

## 3. Prerequisites

**Hardware:**
- A UEFI system (not legacy BIOS)
- TPM 2.0 chip (check: `cat /sys/class/tpm/tpm0/tpm_version_major` should return `2`)
- Secure Boot capable firmware (most systems since ~2012)

**Knowledge:**
- Basic Linux command line comfort
- You don't need to understand everything upfront — each section explains the *why*

**You will need:**
- Arch Linux ISO burned to a USB (`dd` or Ventoy both work)
- A separate machine or phone to read this guide during install (or print it)

---

## 4. Repo Structure

```
.
├── README.md                  ← You are here. The complete step-by-step guide.
├── configs/
│   ├── cmdline                ← Kernel parameters
│   ├── loader.conf            ← systemd-boot loader config
│   ├── mkinitcpio.conf        ← initramfs with systemd + tpm2 hooks
│   ├── pacman-packages.txt    ← Full package list
│   └── snapper/               ← Snapper config templates
├── docs/
│   ├── boot-chain-explained.md
│   ├── tpm2-pcr-policy.md     ← Deep dive on PCR registers and policy binding
│   ├── uki-explained.md       ← What goes into a UKI and why
│   └── recovery.md            ← Full recovery procedures
└── scripts/
    ├── config.sh              ← ALL user variables go here, edit before running
    ├── install.sh             ← Master orchestrator — runs all scripts in order
    ├── 01-partition.sh
    ├── 02-luks-format.sh
    ├── 03-btrfs-setup.sh
    ├── 04-pacstrap.sh
    ├── 05-chroot-setup.sh
    ├── 06-uki-build.sh
    ├── 07-bootloader.sh
    ├── 08-secureboot-keys.sh
    └── 09-tpm2-enroll.sh
```

**To use the scripts**, edit `scripts/config.sh` first, then either run `install.sh`
for a guided full install, or run individual scripts for specific steps.

---

## 5. Pre-Installation

### 5.1 Boot the Arch ISO

Download the latest ISO from [archlinux.org](https://archlinux.org/download/) and
write it to a USB drive:

```bash
# Replace /dev/sdX with your USB drive — double-check this with lsblk
dd bs=4M if=archlinux-*.iso of=/dev/sdX conv=fsync oflag=direct status=progress
```

Boot from the USB. In your UEFI firmware settings, you may need to temporarily
**disable Secure Boot** to boot the Arch ISO (we will set up our own keys later).

### 5.2 Verify UEFI Mode

```bash
cat /sys/firmware/efi/fw_platform_size
```

Should return `64` (64-bit UEFI) or `32`. If the file doesn't exist, you booted in
legacy BIOS mode — go back into your firmware and enable UEFI.

### 5.3 Connect to the Internet

**Ethernet:** Usually works automatically. Verify with `ping archlinux.org`.

**Wi-Fi:**
```bash
iwctl
  station wlan0 scan
  station wlan0 get-networks
  station wlan0 connect YOUR_SSID
  exit
```

### 5.4 Update the System Clock

```bash
timedatectl set-ntp true
timedatectl status   # verify it looks correct
```

---

## 6. Disk Partitioning

Identify your target disk:

```bash
lsblk
# or for NVMe drives specifically:
lsblk -d -o NAME,SIZE,MODEL
```

> **From here on,** the guide uses `/dev/nvme0n1` as the example disk.
> Replace it with your actual disk name everywhere.

We need exactly two partitions:

| Partition | Size | Type | Purpose |
|---|---|---|---|
| `/dev/nvme0n1p1` | 1 GiB | EFI System (ef00) | systemd-boot + UKI |
| `/dev/nvme0n1p2` | Remainder | Linux LUKS (8309) | Encrypted Btrfs root |

```bash
gdisk /dev/nvme0n1
```

Inside gdisk:
```
o        # Create new empty GPT table (confirm with y)
n        # New partition
  1      # Partition number
         # First sector (default)
  +1G    # Last sector
  ef00   # EFI System type

n        # New partition
  2      # Partition number
         # First sector (default)
         # Last sector (default — use rest of disk)
  8309   # Linux LUKS type

w        # Write and exit (confirm with y)
```

Verify:
```bash
lsblk /dev/nvme0n1
gdisk -l /dev/nvme0n1
```

Format the EFI partition:
```bash
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
```

→ Script: `scripts/01-partition.sh`

---

## 7. LUKS2 Encryption

Format the partition with LUKS2. The options here choose strong defaults — `argon2id`
for the key derivation function (resistant to GPU cracking) and `aes-xts-plain64` for
the cipher.

```bash
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha256 \
  --pbkdf argon2id \
  --label cryptroot \
  /dev/nvme0n1p2
```

You will be asked to type `YES` (uppercase) and set a passphrase. **This passphrase is
your recovery fallback.** Store it somewhere safe — a password manager or written down
in a physically secure place. If TPM2 unsealing fails and you lose this passphrase,
your data is gone permanently.

Open (unlock) the LUKS container:
```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
```

The decrypted device is now available at `/dev/mapper/cryptroot`.

→ Script: `scripts/02-luks-format.sh`  
→ See [docs/luks-options-explained.md](docs/luks-options-explained.md) for why these specific options were chosen.

---

## 8. Btrfs Setup

### 8.1 Format and Mount

```bash
mkfs.btrfs -L archroot /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
```

### 8.2 Subvolume Layout

We create separate subvolumes so that snapshots of `@` (root) don't include large,
frequently-changing directories like logs or package cache — keeping snapshots small
and meaningful.

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@snapshots
```

| Subvolume | Mounted at | Snapshotted |
|---|---|---|
| `@` | `/` | Yes — system snapshots |
| `@home` | `/home` | Yes — user data snapshots |
| `@log` | `/var/log` | No — logs excluded |
| `@cache` | `/var/cache` | No — cache excluded |
| `@snapshots` | `/.snapshots` | — stores the snapshots themselves |

Unmount and remount with subvolumes:
```bash
umount /mnt

mount -o noatime,compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt

mkdir -p /mnt/{home,var/log,var/cache,.snapshots,boot}

mount -o noatime,compress=zstd,subvol=@home      /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=zstd,subvol=@log       /dev/mapper/cryptroot /mnt/var/log
mount -o noatime,compress=zstd,subvol=@cache     /dev/mapper/cryptroot /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots

mount /dev/nvme0n1p1 /mnt/boot
```

### 8.3 Mount Options

- `noatime` — disables access time updates on reads, reduces unnecessary writes to SSD
- `compress=zstd` — transparent compression, typically saves 20–40% space with no
  perceptible performance cost on modern hardware

→ Script: `scripts/03-btrfs-setup.sh`

---

## 9. Base System Installation

Install the base system and essential packages:

```bash
pacstrap -K /mnt \
  base base-devel linux linux-firmware \
  btrfs-progs \
  networkmanager \
  systemd-boot \
  tpm2-tools tpm2-tss \
  sbctl \
  neovim \
  man-db man-pages
```

> The full package list including optional tools is at `configs/pacman-packages.txt`.

Generate fstab:
```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab   # review it — make sure all 5 subvolumes and /boot are present
```

→ Script: `scripts/04-pacstrap.sh`

---

## 10. System Configuration (chroot)

Enter the new system:
```bash
arch-chroot /mnt
```

**Timezone:**
```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

**Locale:**
```bash
# Uncomment your locale in /etc/locale.gen, e.g. en_US.UTF-8 UTF-8
nvim /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

**Hostname:**
```bash
echo "your-hostname" > /etc/hostname
```

**Root password:**
```bash
passwd
```

**Create a user:**
```bash
useradd -m -G wheel -s /bin/bash yourusername
passwd yourusername
EDITOR=nvim visudo   # uncomment the %wheel ALL=(ALL:ALL) ALL line
```

**Enable NetworkManager:**
```bash
systemctl enable NetworkManager
```

→ Script: `scripts/05-chroot-setup.sh`

---

## 11. Bootloader — systemd-boot

Install systemd-boot to the EFI partition:
```bash
bootctl install
```

Create the loader configuration at `/boot/loader/loader.conf`:
```ini
default  arch.efi
timeout  5
console-mode max
editor   no
```

> `editor no` prevents anyone from modifying kernel parameters at the boot menu —
> important for security since parameter changes could bypass protections.

We don't create a traditional boot entry file here because the UKI (next section) is
self-contained and discovered automatically by systemd-boot.

→ Config: `configs/loader.conf`  
→ Script: `scripts/07-bootloader.sh`

---

## 12. Unified Kernel Image (UKI)

### 12.1 What is a UKI and Why Use One

A UKI is a single `.efi` file that bundles:
- The kernel
- The initramfs
- The kernel command line
- OS metadata

Because everything is in one signed binary, Secure Boot signatures cover the kernel
command line too — meaning an attacker can't modify boot parameters even if they can
write to the EFI partition. This is the main reason to use UKIs over separate kernel +
initramfs entries.

→ See [docs/uki-explained.md](docs/uki-explained.md) for the full picture.

### 12.2 Configuring mkinitcpio

The kernel command line goes in `/etc/kernel/cmdline` (mkinitcpio bakes this into the
UKI):

```bash
# Replace the UUID with the UUID of your LUKS partition (/dev/nvme0n1p2)
blkid -s UUID -o value /dev/nvme0n1p2
```

```bash
# /etc/kernel/cmdline
rd.luks.name=YOUR-LUKS-UUID=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ rw quiet
```

Edit `/etc/mkinitcpio.conf`. The critical part is the HOOKS line — we use the
`systemd`-based hook set which supports TPM2 unlocking:

```ini
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)
```

Also enable UKI output in `/etc/mkinitcpio.d/linux.preset`:

```ini
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default')

default_uki="/boot/EFI/Linux/arch-linux.efi"
default_options="--splash=/usr/share/systemd/bootctl/splash-arch.bmp"
```

→ Configs: `configs/mkinitcpio.conf`, `configs/cmdline`

### 12.3 Building the UKI

Create the output directory and build:
```bash
mkdir -p /boot/EFI/Linux
mkinitcpio -p linux
```

Verify the UKI was created:
```bash
ls -lh /boot/EFI/Linux/arch-linux.efi
```

→ Script: `scripts/06-uki-build.sh`

---

## 13. Secure Boot

### 13.1 Enrolling Your Own Keys with sbctl

`sbctl` manages your own Platform Key (PK), Key Exchange Key (KEK), and Signature
Database (db). This replaces the default OEM keys with keys only you control.

Check current Secure Boot state:
```bash
sbctl status
```

Create your keys:
```bash
sbctl create-keys
```

Enroll them (the `--microsoft` flag includes Microsoft's keys alongside yours — needed
to keep booting Windows or UEFI Option ROMs that are Microsoft-signed):
```bash
sbctl enroll-keys --microsoft
```

> If you're on a system with only Linux and want maximum control, you can omit
> `--microsoft`. On most laptops this will break firmware updates and some NVMe
> drivers — keep `--microsoft` unless you know your hardware doesn't need it.

### 13.2 Signing the UKI

```bash
sbctl sign -s /boot/EFI/Linux/arch-linux.efi
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
```

The `-s` flag saves these paths to sbctl's database so they get re-signed automatically.

Verify everything that needs signing is signed:
```bash
sbctl verify
```

### 13.3 Automating Re-signing on Kernel Updates

Every time a new kernel is installed, `mkinitcpio` regenerates the UKI and the
signature becomes invalid. A pacman hook handles this automatically.

Create `/etc/pacman.d/hooks/99-secureboot.hook`:
```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux

[Action]
Description = Signing new UKI for Secure Boot...
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
```

This runs `sbctl sign-all` after every kernel update, re-signing all saved paths.

→ Script: `scripts/08-secureboot-keys.sh`

---

## 14. TPM2 Enrollment

### 14.1 Understanding PCR Policies

PCR (Platform Configuration Register) values are measurements taken by the firmware
during boot. TPM2 will only release the LUKS key if the PCR values at unlock time match
what was recorded at enrollment time — if anything in the measured boot chain changes,
the TPM refuses to unlock and you fall back to your passphrase.

We bind to **PCR 7 + PCR 11**:

| PCR | Measures | Why we use it |
|---|---|---|
| 7 | Secure Boot state and keys | Ensures Secure Boot is on and our keys are loaded |
| 11 | The UKI that was loaded | Ensures the exact kernel+initramfs hasn't been tampered with |

→ See [docs/tpm2-pcr-policy.md](docs/tpm2-pcr-policy.md) for the full explanation of all PCR registers and why other PCRs (like 0-6) are often avoided.

### 14.2 Enrolling the LUKS Volume

Make sure you're booted into your new installed system with Secure Boot active before
running this, otherwise the PCR values won't match your intended boot environment.

```bash
# Verify TPM2 is present and accessible
systemd-cryptenroll --tpm2-device=list

# Enroll — you'll be prompted for your LUKS passphrase to authorize the change
systemd-cryptenroll \
  --tpm2-device=auto \
  --tpm2-pcrs=7+11 \
  /dev/nvme0n1p2
```

Tell the initramfs to use TPM2 for unlocking by updating `/etc/crypttab.initramfs`:

```
cryptroot UUID=YOUR-LUKS-UUID none tpm2-device=auto
```

Rebuild the UKI and re-sign:
```bash
mkinitcpio -p linux
sbctl sign-all
```

### 14.3 Testing and Fallback

Reboot. The system should unlock automatically without asking for a passphrase.

If TPM2 unsealing fails (wrong PCRs, Secure Boot state changed, etc.), `systemd-cryptsetup`
will automatically fall back to prompting for your LUKS passphrase. **This is expected
and correct behaviour** — it means your fallback is working.

After any of these events you will need to re-run the enrollment command in 14.2:
- Kernel update (new UKI changes PCR 11)
- Secure Boot key changes (changes PCR 7)
- Firmware update (may change PCR 7)

→ Script: `scripts/09-tpm2-enroll.sh`  
→ See [docs/tpm2-pcr-policy.md](docs/tpm2-pcr-policy.md)

---

## 15. Snapper — Btrfs Snapshots

Install Snapper and create configs for root and home:

```bash
pacman -S snapper snap-pac

snapper -c root create-config /
snapper -c home create-config /home
```

Edit `/etc/snapper/configs/root` to set retention limits (adjust to your preference):
```ini
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="2"
TIMELINE_LIMIT_MONTHLY="1"
TIMELINE_LIMIT_YEARLY="0"
```

Enable the timers:
```bash
systemctl enable --now snapper-timeline.timer
systemctl enable --now snapper-cleanup.timer
```

`snap-pac` is a pacman hook that automatically creates pre/post snapshots around every
pacman transaction — meaning you can always roll back a bad update.

---

## 16. Post-Installation Checklist

Before rebooting into your final system, verify:

- [ ] `lsblk` shows correct partition layout
- [ ] `cat /mnt/etc/fstab` has all 5 Btrfs subvolumes and `/boot`
- [ ] `cat /mnt/etc/kernel/cmdline` contains the correct LUKS UUID
- [ ] `/mnt/boot/EFI/Linux/arch-linux.efi` exists
- [ ] `sbctl verify` shows all required files are signed
- [ ] `sbctl status` shows Secure Boot is set up (will be enabled after reboot)
- [ ] `/mnt/etc/crypttab.initramfs` contains the correct LUKS UUID

Exit chroot and reboot:
```bash
exit
umount -R /mnt
reboot
```

In your firmware settings, **enable Secure Boot** before the system boots.

---

## 17. Recovery Guide

If something goes wrong and you can't boot:

1. Boot from the Arch ISO
2. Open the LUKS container: `cryptsetup open /dev/nvme0n1p2 cryptroot`
3. Mount the Btrfs root: `mount -o subvol=@ /dev/mapper/cryptroot /mnt`
4. Mount the rest: `mount /dev/nvme0n1p1 /mnt/boot` (and other subvolumes)
5. Chroot: `arch-chroot /mnt`

From here you can rebuild the UKI, re-sign, re-enroll TPM2, or roll back a Snapper snapshot.

→ See [docs/recovery.md](docs/recovery.md) for detailed recovery scenarios.

---

## 18. Troubleshooting

**TPM2 won't unseal / always asks for passphrase**
- Did you enroll *after* enabling Secure Boot? PCR 7 changes between Secure Boot on/off.
- Did the kernel update without re-enrolling? PCR 11 now differs.
- Fix: boot with passphrase → wipe old TPM2 slot → re-enroll.
  ```bash
  systemd-cryptenroll --wipe-slot=tpm2 /dev/nvme0n1p2
  # then re-run the enrollment command from section 14.2
  ```

**`sbctl verify` shows unsigned files**
- Run `sbctl sign-all` to re-sign all saved paths.

**System won't boot after kernel update**
- The UKI may not have been regenerated. Boot from ISO, chroot, run `mkinitcpio -p linux` and `sbctl sign-all`.

**Secure Boot enrollment fails in firmware**
- Some firmware requires you to clear existing keys first (enter "Setup Mode") before enrolling new ones. Look for "Reset to Setup Mode" or "Clear Secure Boot Keys" in your UEFI settings.

**Btrfs errors at mount**
- Run `btrfs check --readonly /dev/mapper/cryptroot` to check filesystem health.

---

## Acknowledgements

- [Arch Wiki — dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
- [Arch Wiki — Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [Arch Wiki — Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Arch Wiki — TPM2 and systemd-cryptenroll](https://wiki.archlinux.org/title/Systemd-cryptenroll)

---

*Pull requests and issues welcome. If something in this guide is wrong or has changed,
please open an issue.*
