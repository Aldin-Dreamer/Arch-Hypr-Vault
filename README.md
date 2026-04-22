<h1 align="center">
  <a href="https://github.com/Aldin-Dreamer/Arch-Hypr-Vault">
    <!-- Please provide path to your logo here -->
    <img src="docs/images/logo.svg" alt="Logo" width="100" height="100">
  </a>
</h1>

<div align="center">
  Arch Linux — Seamless LUKS Encrypted Boot with TPM2 Auto-Unlock and Secure Boot
  <br />
  <br />
  <a href="https://github.com/Aldin-Dreamer/Arch-Hypr-Vault/issues/new?assignees=&labels=bug&template=01_BUG_REPORT.md&title=bug%3A+">Report a Bug</a>
  ·
  <a href="https://github.com/GITHUB_USERNAME/REPO_SLUG/issues/new?assignees=&labels=question&template=04_SUPPORT_QUESTION.md&title=support%3A+">Ask a Question</a>
</div>

<div align="center">
<br />

[![Project license](https://img.shields.io/github/license/Aldin-Dreamer/Arch-Hypr-Vault.svg?style=flat-square)](LICENSE)
[![Pull Requests welcome](https://img.shields.io/badge/PRs-welcome-ff69b4.svg?style=flat-square)](https://github.com/Aldin-Dreamer/Arch-Hypr-Vault/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22)
[![code with love by GITHUB_USERNAME](https://img.shields.io/badge/%3C%2F%3E%20with%20%E2%99%A5%20by-Aldin--Dreamer-ff1414.svg?style=flat-square)](https://github.com/Aldin-Dreamer)

</div>

---

An installation guide for security focused users who want a seamlessly encrypted system with LUKS encryption, TPM2 auto unlock and secure boot. This guide is meant to be used alongside the official Arch Wiki Installation guide. This guide will cover how the setup works and how to replicate it yourself and what the automated install scripts do. Filesystem and tooling choices are also made with day-to-day usability in mind — such as Btrfs for snapshot-based rollbacks.

> ⚠️ **Warning:** This process involves disk partitioning and will erase all data
> on the target drive. Back up anything important before proceeding. There is also
> a real risk of bricking your system — read the entire guide at least once before
> running any commands. The automated scripts may also have bugs or behave
> differently across hardware.

> 💡 This guide doubles as a full documentation of my personal setup — dotfiles,
> desktop configuration and all. I recommend you to pick and choose what's relevant to
> you rather than running the full install script, which will replicate my setup
> exactly. Happy installing! 🎉

---

<details open="open">
<summary>Table of Contents</summary>

- [What This Setup Achieves](#1-what-this-setup-achieves)
- [How It All Fits Together](#2-how-it-all-fits-together)
- [Prerequisites](#3-prerequisites)
- [Repo Structure](#4-repo-structure)
- [Pre-Installation](#5-pre-installation)
- [Disk Partitioning](#6-disk-partitioning)
- [LUKS2 Encryption](#7-luks2-encryption)
- [Btrfs Setup](#8-btrfs-setup)
- [Base System Installation](#9-base-system-installation)
- [System Configuration](#10-system-configuration)
- [Bootloader — systemd-boot](#11-bootloader--systemd-boot)
- [Unified Kernel Image (UKI)](#12-unified-kernel-image-uki)
- [Secure Boot](#13-secure-boot)
- [TPM2 Enrollment](#14-tpm2-enrollment)
- [Snapper — Btrfs Snapshots](#15-snapper--btrfs-snapshots)
- [Post-Installation Checklist](#16-post-installation-checklist)
- [Recovery Guide](#17-recovery-guide)
- [Troubleshooting](#18-troubleshooting)
- [Desktop Setup](#desktop-setup)
- [Contributing](#contributing)
- [Authors & Contributors](#authors--contributors)
- [License](#license)
- [Acknowledgements](#acknowledgements)

</details>

---

## 1. What This Setup Achieves

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
| [ ] | [ ] |

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

<!-- Credit the Arch Wiki pages, blog posts, or repos that helped you -->

-
-
