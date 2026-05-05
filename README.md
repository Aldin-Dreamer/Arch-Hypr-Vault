<h1 align="center">
  <a href="https://github.com/Aldin-Dreamer/Arch-Hypr-Vault">
    <img src="docs/images/logo.svg" alt="Logo" width="100" height="100">
  </a>
</h1>

<div align="center">
  Arch Linux — Seamless LUKS Encrypted Boot with TPM2 Auto-Unlock and Secure Boot
  <br />
  <br />
  <a href="https://github.com/Aldin-Dreamer/Arch-Hypr-Vault/issues/new?assignees=&labels=bug&template=01_BUG_REPORT.md&title=bug%3A+">Report a Bug</a>

<div align="center">
<br />

[![Project license](https://img.shields.io/github/license/Aldin-Dreamer/Arch-Hypr-Vault.svg?style=flat-square)](LICENSE)
[![Pull Requests welcome](https://img.shields.io/badge/PRs-welcome-ff69b4.svg?style=flat-square)](https://github.com/Aldin-Dreamer/Arch-Hypr-Vault/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22)
[![code with love by GITHUB_USERNAME](https://img.shields.io/badge/%3C%2F%3E%20with%20%E2%99%A5%20by-Aldin--Dreamer-ff1414.svg?style=flat-square)](https://github.com/Aldin-Dreamer)

</div>

---

An installation guide for security focused users who want a seamlessly encrypted system with LUKS encryption, TPM2 auto unlock and secure boot. This guide is meant to be used alongside the official ArchWiki Installation guide. This guide will cover how the setup works and how to replicate it yourself. Filesystem and tooling choices are also made with day-to-day usability in mind — such as Btrfs for snapshot-based rollbacks.

> ⚠️ **Warning:** This process involves disk partitioning and will erase all data
> on the target drive. Back up anything important before proceeding. There is also
> a real risk of bricking your system — read the entire guide at least once before
> running any commands. The automated scripts may also have bugs or behave
> differently across hardware.

---

<details open="open">
<summary>Table of Contents</summary>

 [Repo Structure](#1-repo-structure)<br>
 [What This Setup Achieves](#2-what-this-setup-achieves)<br>
 [Prerequisites](#3-prerequisites)<br>
 [How It All Fits Together](#4-how-it-all-fits-together)<br>
 [Disk Partitioning](#5-disk-partitioning)<br>
 [LUKS2 Encryption](#6-luks2-encryption)<br>
 [Btrfs Setup](#7-btrfs-setup)<br>
 [Base System Installation](#8-base-system-installation)<br>
 [System Configuration](#9-system-configuration)<br>
 [Bootloader — systemd-boot](#10-bootloader--systemd-boot)<br>
 [Unified Kernel Image (UKI)](#11-unified-kernel-image-uki)<br>
 [Secure Boot](#12-secure-boot)<br>
 [TPM2 Enrollment](#13-tpm2-enrollment)<br>
 [Snapper — Btrfs Snapshots](#14-snapper--btrfs-snapshots)<br>
 [Post-Installation Checklist](#15-post-installation-checklist)<br>
 [Recovery Guide](#16-recovery-guide)<br>
 [Troubleshooting](#17-troubleshooting)<br>
 [Desktop Setup](#desktop-setup)<br>
 [Contributing](#contributing)<br>
 [Authors & Contributors](#authors--contributors)<br>
 [License](#license)<br>
 [Acknowledgements](#acknowledgements)<br>
 
</details>

---
</div>

## 1. Repo Structure

```
Arch-Hypr-Vault/
├── .gitignore
├── LICENSE
├── README.md
├── RICE.md
├── .config/
└── docs/
    └── images/
        └── logo.svg
```

---
<div align="center">

## 2. What This Setup Achieves

**Security & Boot**

| Feature | Implementation |
|---|---|
| Full disk encryption | LUKS2 |
| Automatic unlock at boot | TPM2 via `systemd-cryptenroll` |
| Fallback unlock | LUKS passphrase |
| Protection against Evil Maid attacks | Secure Boot with personal keys via `sbctl` |
| Bootloader | `systemd-boot` |
| Unified boot image | UKI — kernel + initramfs + cmdline in one signed `.efi` |

**Desktop & Personal Choices** *(swap these out for your own preferences)*

| Feature | Implementation |
|---|---|
| Filesystem | Btrfs |
| Snapshot support | Snapper |
| Windows Manager | Hyprland |

>🔒**Security Scope:** This setup will protect your data at rest i.e, if your device gets stolen or is physically tampered with by malicious actors. It wont however protect your setup when it is powered on and running, so you are still vulnerable to attacks from the internet, malware and even when your laptop is stolen while it is powered on. For that you need additional measure such as a firewall, keeping you system updated and not leave it powered on in public places.


---

## 3. Prerequisites

**You will need:**
<div align="left">
  <ul>
    <li>A UEFI system (legacy BIOS doesn't support this setup)</li>
    <li>A TPM2 chip (You can check if you have it <a href="https://wiki.archlinux.org/title/Trusted_Platform_Module#Checking_TPM_support">here</a>)</li>
    <li>You are expected to have read the <a href="https://wiki.archlinux.org/title/Installation_guide">ArchWiki Installation Guide</a> and meet its prerequisites.</li>
    <li>A lot of free time especially if you are new. Dont rush this.</li>
  </ul>
</div>

> 📝 **Note:** I will be using the ArchWiki syntax so commands prefixed with `#` are run as root. In the Arch ISO you are already root so no `sudo` is needed. After installation, use `sudo` where required.

---

## 4. How It All Fits Together

```mermaid
flowchart TD
    A[Power On] --> B[UEFI]
    B --> |Runs|C[POST]
    C --> D{Secure boot}
    D --> |Fail|E[Boot Denied]
    D --> |Pass|F[systemd-boot]
    F --> G[UKI]
```

<!-- To Be Elaborated -->
<div align="left">
  
**UEFI** — Initializes the hardware and reads the boot entries from NVRAM to determine which EFI partition.

**POST** — POST stands for power on self test. It checks if the hardware is working properly before booting.

**Secure Boot** — Checks cryptographic signatures on the bootloader and UKI before 
allowing them to run. If any modification has been identified, it will deny boot.

**systemd-boot** — 

**Unified Kernel Image (UKI)** — 
he following steps will wipe the dis
**TPM2 PCR Check** — 
**Btrfs Mount** — 
</div>

---

## 5. Disk Partitioning

Before we begin, it is important to visualize how disk space and partitions work: <br><br>
**[ Partition 1 | Partition 2 | *Free Space* | Partition 3 | *Free space* ]** <br><br>
Given this partition layout, there is an important distinction to make here - the memory is discontinuous. If you want to increase the size of partition 2 in the future, you can only increase it upto the space in front of it, i.e, you cannot use the space available after partition 3 to increase the size of partition 2. This is why we make EFI partition first and root partition last so we can extend it if needed or even shrink it (Though i must add there is a complication in extending and shrinking partitions in this specific setup which will be explained in the LUKS encryption part).

> There is a way to move partitions, but it is risky and involves rewriting the entire contents of the partition to where you want to move it. This takes a long time and there is a significant risk of data loss.<br>
> 📖 **Further reading:** [Moving Partitions — ArchWiki](https://wiki.archlinux.org/title/Fdisk#Moving_partitions)

 5.1 Partition Sceheme
 ---

Follow the Arch Wiki Installation Guide till <a href="https://wiki.archlinux.org/title/Installation_guide#Update_the_system_clock">Updating the system clock</a>

The recommended partition strategy for this setup is:
| Mount Point | Partition type | Recommended Size |
|---|---|---|
| /boot | EFI Partition | 1 GiB |
| / | Root Partition | Remainder of the space. At least 23-32 GiB |

**EFI Partition -** The EFI partition is where the Unified Kernel Image(UKI) lives with the kernel, initramfs and microcode. It is recommended to use 1 GiB for future-proofing, so if you can spare it - do it. The UKI is pretty big (~100-150 MiB) so if you plan to put multiple kernel entries, the 1 GiB headroom will be useful. If you have space constraints, you can use 512 MiB instead.

**Root Partition -** This is where the main filesystem lives and hence where you will store most of your data. The size for this partition will generally be whatever you have remaining, so choose a size as per your use case.
<!-- Mention Zram as an alternative -->
> 📝**Note:** A swap partition is not recommended. An unencrypted swap partition will hold onto data when you shut down, so anything that the kernel places into the swap file during normal operation will be saved unencrypted. If you do need swap, I recommend to use a swap file instead - it will lie under the LUKS encryption and provide the functionality of swap without compromising security. If you still require a swap partition, I recommend reading: [Swap Encryption — ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption)

 5.2 Creating the partitions
 ---

> ⚠️**Warning:** The following steps will wipe the disk!! Backup anything you wish to save.

The steps vary for users who are dual booting. For dual booting with windows (other dual booters are also recommended to read this), see [docs/dual-boot-windows.md](docs/dual-boot-windows.md). The following is for users on a new disk or users who are gonna wipe the entire drive for a fresh install:

Before starting with the partitoning,run: 
<div align="left">
  
```bash
# lsblk
```
</div>

You can identify which block device your disk was assigned to (Most commonly it is /dev/sda or /dev/nvme0n1). In the following steps replace '*/dev/[device]*' with your block device.<br><br>

💡**Optional — 4096 Native Sector Size:** Most solid state drives (SSD) report their logical sector size as 512 bytes, even though they use larger sector size physically. If your SSD support 4096 bytes sector size, it is recommended to format it before partitioning to improve both the performance and lifespan of your SSD. Check if it supported by running:
>⚠️**Warning:** This specific step will wipe your entire disk and not just partitions, I am warning you again to backup anything you wish to save.
>
>If you are using a USB attached external drive, read [Advanced Format — ArchWiki](https://wiki.archlinux.org/title/Advanced_Format#Advanced_Format_hard_disk_drives) before deciding to change your sector size. There are some NVMe SSD that report they support 4096 bytes sectors but they encounter unstability especially under random heavy read load. If encountering this issue, simply revert back to 512 byte sector configuration.
<div align="left">
  
  ```bash
# nvme id-ns -H /dev/[device] | grep "Relative Performance"
---------------------------------------------------------------------------------------------------------------------------------

LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0 Best (in use)
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0 Best 
```
</div>
If you see a format with 4096 bytes, thats your native sector size. If it is supported but shows 512 bytes format is in use, you can change it by running:
<div align="left">

```bash
# nvme format --lbaf=1 /dev/[device]
----------------------------------------------------------------------------------------------------------------------------------
You are about to format [device], namespace 0x1.
WARNING: Format may irrevocably delete this device's data.
You have 10 seconds to press Ctrl-C to cancel this operation.

Use the force [--force] option to suppress this warning.
Sending format operation ... 
Success formatting namespace:
```
</div>
Then verify that your logical sector size changed:
<div align="left">

  ```bash
# nvme id-ns -H /dev/[device] | grep "Relative Performance"
---------------------------------------------------------------------------------------------------------------------------------

LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0 Best 
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0 Best (in use) 
```
</div>


Now to begin partitioning your chosen drive,run:
<div align="left">
  
```bash
# fdisk /dev/[device]
```
</div>

You will enter an interactive command-line interface (CLI), you can type 'm' to get the full list of commands.
<!-- TO BE REWORKED -->
<div align="left">
  
  - Type 'p' to print the current partition table. It will show something like:
  
  ```bash
  
Disk /dev/[device]: [size] GiB, [bytes] bytes, [total_sectors] sectors
Disk model: [Manufacturer_model_name]              
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: [gpt/dos]
Disk identifier: [UNIQUE-ID-OR-GUID]

Device                                 Start        End        Sectors   Size       Type
/dev/[device][EFI_partition_number]    2048       [sector]     [count]  [size] [Partition_Type]
/dev/[device][root_partition_number]  [sector]    [sector]     [count]  [size] [Partition_Type]
  ```
- If you do not have partitons, only the disk metadata will be shown. If your disklabel type is empty or dos, type 'g' to create a new gpt partition table.
- To delete you existing partition, type 'd', then the number of the partition you want to delete.
- To create new partitions, type 'n', then the number you want to assign the partition then the space
- To create the EFI partition, type 'n' -> 1 -> enter -> +1G
- To create the root partition, type 'n' -> 2 -> enter -> enter (this will assign rest of the available space to root)
- Before saving the changes, type 'p' again to see the new partition made
  ```bash
  Disk /dev/[device]: [size] GiB, [bytes] bytes, [total_sectors] sectors
  Disk model: [Manufacturer_model_name]              
  Units: sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 4096 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes
  Disklabel type: [gpt/dos]
  Disk identifier: [UNIQUE-ID-OR-GUID]
  
  Device                                 Start        End        Sectors   Size       Type
  /dev/[device][EFI_partition_number]    2048       [sector]     [count]  [size]   EFI System
  /dev/[device][root_partition_number]  [sector]    [sector]     [count]  [size]  Linux filesystem
  ```
- Type 'w' to save the changes and exit.
</div>


---

## 6. LUKS2 Encryption

Once the partitions have been created, each newly created partition must be formatted with an appropriate file system. 
For the EFI partition, we will format it with FAT32 filesystem. This is the recommended option as the choice of filesystem needs to adhere to [UEFI specifications](https://uefi.org/specs/UEFI/2.11/13_Protocols_Media_Access.html#file-system-format-1). To create it, run:
>⚠️**Warning:** Only format the EFI system partition if you created it during the partitioning step. If there already was an EFI system partition on disk beforehand, reformatting it can destroy the boot loaders of other installed operating systems.
<div align="left">

```bash
# mkfs.fat -F 32 /dev/[device][EFI_partition_number]
```
</div>

For the root partition, before we create the filesystem, we need to encrypt it. To create the LUKS volume, run:
>📝**Note:** In the previous section, if you did not do the optional step of changing the logical sector size of you SSD, omit the *--sector-size 4096* in the below command.
<div align="left">
  
```bash
# cryptsetup luksFormat --sector-size 4096 /dev/[device][root_partition_number]
```
</div>
You will be prompted to enter a password for unlocking the encryption. This will be your fallback passphrase if the TPM2 auto-unlock fails.

>❗**IMPORTANT:** Make sure to use a sufficiently secure password. Even though the keyslot will be wiped later, SSD wear-leveling can cause it to persist after removal for an indefinite amount of time. Also make sure to physically write this down or save it somewhere secure, if you lose this passphrase and TPM2 auto-unlock fails — you will be locked out of your system forever.

Now to create the filesystem for root partition, first open the LUKS volume to access the root partition. Run the command and enter the password:
<div align="left">
  
```bash
# cryptsetup open /dev/[device][root_partition_number] root
```
</div>

You can verify if your root partition has been successfully encrypted by running lsblk. Instead of the root partition you created, you will see /dev/mapper/root:
<div align="left">

```bash
lsblk
```
</div>

Then create a filesystem on the unlocked root partition. This is where you can choose your preferred filesystem. If you are following this guide exactly, it would be the btrfs filesystem. 

> 📖 **Filesystem Reference:** Watch this [beginner friendly overview](https://www.youtube.com/watch?v=4kMt5RtDZ7g) or read the [File Systems — ArchWiki](https://wiki.archlinux.org/title/File_systems#Types_of_file_systems) for a detailed comparison. If you choose a different filesystem, replace mkfs.btrfs in the command below with the appropriate command for your choice.
<div align="left">
  
```bash
# mkfs.btrfs /dev/mapper/root
```
</div>
Then mount both the root partition and EFI partitions:
<div align="left">
  
```bash
# mount /dev/mapper/root /mnt
# mount --mkdir /dev/[device][EFI_partition_number] /mnt/boot
```
</div>

---

## 7. Btrfs Setup

### 7.1 Format and Mount

### 7.2 Subvolume Layout

<!-- Which subvolumes are you creating and why?
     A table of subvolume → mount point → snapshotted yes/no is useful here. -->

### 7.3 Mount Options

<!-- What mount options are you using (noatime, compress=zstd, etc.) and what does each one do? -->

---

## 8. Base System Installation

<!-- pacstrap command with your package list.
     genfstab — and remind the reader to verify it looks correct. -->

---

## 9. System Configuration

<!-- Everything inside arch-chroot:
     timezone, locale, hostname, root password, user creation, sudo, services to enable -->

---

## 10. Bootloader — systemd-boot

<!-- bootctl install, loader.conf, and why you chose the options you did -->

---

## 11. Unified Kernel Image (UKI)

### 11.1 What is a UKI and Why Use One

<!-- Explain what a UKI bundles and the security benefit over separate kernel + initramfs entries.
     Link to docs/uki-explained.md for the deep dive. -->

### 11.2 Configuring mkinitcpio

<!-- /etc/kernel/cmdline — how to find your LUKS UUID and what parameters go here.
     /etc/mkinitcpio.conf — the HOOKS line, especially why systemd hooks are used.
     /etc/mkinitcpio.d/linux.preset — enabling UKI output. -->

### 11.3 Building the UKI

---

## 12. Secure Boot

### 12.1 Enrolling Your Own Keys with sbctl

<!-- sbctl create-keys and enroll-keys.
     Note on --microsoft flag and when to include/omit it. -->

### 12.2 Signing the UKI

### 12.3 Automating Re-signing on Kernel Updates

<!-- The pacman hook. Show the hook file and explain when it runs. -->

---

## 13. TPM2 Enrollment

### 13.1 Understanding PCR Policies

<!-- Which PCRs are you binding to and why?
     What does it mean when PCR values change?
     Link to docs/tpm2-pcr-policy.md for the deep dive. -->

### 13.2 Enrolling the LUKS Volume

<!-- systemd-cryptenroll command.
     Updating /etc/crypttab.initramfs.
     Rebuilding and re-signing the UKI after enrollment. -->

### 13.3 Testing and Fallback

<!-- What should happen on a successful reboot?
     What does fallback to passphrase look like, and is that expected?
     When does the reader need to re-enroll? -->

---

## 14. Snapper — Btrfs Snapshots

<!-- snapper create-config, retention settings, enabling timers.
     Mention snap-pac for automatic pre/post snapshots around pacman. -->

---

## 15. Post-Installation Checklist

<!-- A checklist the reader runs through before their first reboot.
     Each item should be something that can actually be verified with a command. -->

- [ ]
- [ ]
- [ ]

---

## 16. Recovery Guide

<!-- What to do if the system won't boot.
     The basic steps: boot ISO → open LUKS → mount Btrfs → chroot.
     Link to docs/recovery.md for detailed scenarios. -->

---

## 17. Troubleshooting

<!-- Common failure points and their fixes.
     Add to this as you hit issues during your own installs. -->

**[Problem]**
> Cause and fix

**[Problem]**
> Cause and fix

---

## Desktop Setup

For the Hyprland rice and desktop configuration see [RICE.md](RICE.md)

---

## Contributing

Contributions, issues and feature requests are welcome. Feel free to check the [issues page](https://github.com/GITHUB_USERNAME/REPO_SLUG/issues) if you want to contribute.

---

## Authors & Contributors

The original setup of this repository is by [Aldin-Dreamer](https://github.com/Aldin-Dreamer).

For a full list of all authors and contributors, see [the contributors page](https://github.com/GITHUB_USERNAME/REPO_SLUG/contributors).

---

## License

This project is licensed under the **MIT license**.

See [LICENSE](LICENSE) for more information.

---

## Acknowledgements


-
-
