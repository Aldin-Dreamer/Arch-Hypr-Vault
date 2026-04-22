# Arch Linux — [Your Setup Title Here]

[Brief description of what this guide covers and who it's for]

> **⚠️ Warning:** [Add your warnings here — data loss, disk wipe, etc.]

---

## Table of Contents

1. [What This Setup Achieves](#1-what-this-setup-achieves)
2. [How It All Fits Together](#2-how-it-all-fits-together)
3. [Prerequisites](#3-prerequisites)
4. [Repo Structure](#4-repo-structure)
5. [Pre-Installation](#5-pre-installation)
6. [Disk Partitioning](#6-disk-partitioning)
7. [LUKS2 Encryption](#7-luks2-encryption)
8. [Btrfs Setup](#8-btrfs-setup)
9. [Base System Installation](#9-base-system-installation)
10. [System Configuration](#10-system-configuration)
11. [Bootloader — systemd-boot](#11-bootloader--systemd-boot)
12. [Unified Kernel Image (UKI)](#12-unified-kernel-image-uki)
13. [Secure Boot](#13-secure-boot)
14. [TPM2 Enrollment](#14-tpm2-enrollment)
15. [Snapper — Btrfs Snapshots](#15-snapper--btrfs-snapshots)
16. [Post-Installation Checklist](#16-post-installation-checklist)
17. [Recovery Guide](#17-recovery-guide)
18. [Troubleshooting](#18-troubleshooting)

---

## 1. What This Setup Achieves

<!-- A table works well here. What feature does each component provide? -->

| Feature | Implementation |
|---|---|
| | |
| | |

---

## 2. How It All Fits Together

<!-- Explain the boot chain from power-on to desktop.
     A text diagram or ASCII art of the sequence is really helpful here.
     The goal is: a reader should understand WHY each piece exists before touching a command. -->

---

## 3. Prerequisites

<!-- What hardware is required? What should the reader already know?
     What do they need to have prepared before starting? -->

**Hardware:**
-

**Knowledge:**
-

**You will need:**
-

---

## 4. Repo Structure

<!-- Show the folder/file tree and briefly describe what each part is for -->

```
.
├── README.md
├── configs/
├── docs/
└── scripts/
```

---

## 5. Pre-Installation

### 5.1 Boot the Arch ISO

### 5.2 Verify UEFI Mode

### 5.3 Connect to the Internet

### 5.4 Update the System Clock

---

## 6. Disk Partitioning

<!-- What partitions are needed, what size, what type?
     A table showing partition / size / purpose is useful here.
     Show the exact commands. Warn the reader to double-check their disk name. -->

---

## 7. LUKS2 Encryption

<!-- The format command and the options you chose — explain briefly why each option matters.
     How do you open the container after formatting?
     What should the reader do with their passphrase? -->

---

## 8. Btrfs Setup

### 8.1 Format and Mount

### 8.2 Subvolume Layout

<!-- Which subvolumes are you creating and why?
     A table of subvolume → mount point → snapshotted yes/no is useful here. -->

### 8.3 Mount Options

<!-- What mount options are you using (noatime, compress=zstd, etc.) and what does each one do? -->

---

## 9. Base System Installation

<!-- pacstrap command with your package list.
     genfstab — and remind the reader to verify it looks correct. -->

---

## 10. System Configuration

<!-- Everything inside arch-chroot:
     timezone, locale, hostname, root password, user creation, sudo, services to enable -->

---

## 11. Bootloader — systemd-boot

<!-- bootctl install, loader.conf, and why you chose the options you did -->

---

## 12. Unified Kernel Image (UKI)

### 12.1 What is a UKI and Why Use One

<!-- Explain what a UKI bundles and the security benefit over separate kernel + initramfs entries.
     Link to docs/uki-explained.md for the deep dive. -->

### 12.2 Configuring mkinitcpio

<!-- /etc/kernel/cmdline — how to find your LUKS UUID and what parameters go here.
     /etc/mkinitcpio.conf — the HOOKS line, especially why systemd hooks are used.
     /etc/mkinitcpio.d/linux.preset — enabling UKI output. -->

### 12.3 Building the UKI

---

## 13. Secure Boot

### 13.1 Enrolling Your Own Keys with sbctl

<!-- sbctl create-keys and enroll-keys.
     Note on --microsoft flag and when to include/omit it. -->

### 13.2 Signing the UKI

### 13.3 Automating Re-signing on Kernel Updates

<!-- The pacman hook. Show the hook file and explain when it runs. -->

---

## 14. TPM2 Enrollment

### 14.1 Understanding PCR Policies

<!-- Which PCRs are you binding to and why?
     What does it mean when PCR values change?
     Link to docs/tpm2-pcr-policy.md for the deep dive. -->

### 14.2 Enrolling the LUKS Volume

<!-- systemd-cryptenroll command.
     Updating /etc/crypttab.initramfs.
     Rebuilding and re-signing the UKI after enrollment. -->

### 14.3 Testing and Fallback

<!-- What should happen on a successful reboot?
     What does fallback to passphrase look like, and is that expected?
     When does the reader need to re-enroll? -->

---

## 15. Snapper — Btrfs Snapshots

<!-- snapper create-config, retention settings, enabling timers.
     Mention snap-pac for automatic pre/post snapshots around pacman. -->

---

## 16. Post-Installation Checklist

<!-- A checklist the reader runs through before their first reboot.
     Each item should be something that can actually be verified with a command. -->

- [ ]
- [ ]
- [ ]

---

## 17. Recovery Guide

<!-- What to do if the system won't boot.
     The basic steps: boot ISO → open LUKS → mount Btrfs → chroot.
     Link to docs/recovery.md for detailed scenarios. -->

---

## 18. Troubleshooting

<!-- Common failure points and their fixes.
     Add to this as you hit issues during your own installs. -->

**[Problem]**
> Cause and fix

**[Problem]**
> Cause and fix

---

## Acknowledgements

<!-- Credit the Arch Wiki pages, blog posts, or repos that helped you -->

-
-

---

*[Any closing note — contributions welcome, license, etc.]*
