# Dual Booting with Windows

> 💡 This guide is a supplement to the main [README.md](../README.md). Complete the 
> steps here before continuing with the partitioning section.

---

## Before You Begin

<!-- What should the reader have already done / checked before starting this.
     Backup warnings, BitLocker warnings, Fast Boot warnings etc. -->

---

## Preparing Windows

### Disable Fast Boot

<!-- Why Fast Boot is a problem for dual booting and how to disable it -->

### Disable BitLocker

<!-- Why BitLocker needs to be addressed before partitioning and how to suspend/disable it -->

### Shrink the Windows Partition

<!-- How to shrink from within Windows Disk Management.
     How much space to leave for Windows, how much to free for Arch. -->

---

## Partitioning

<!-- How the partition layout looks in a dual boot scenario.
     Explain that the Windows EFI partition (p1) will be reused for Arch — 
     no need to create a new EFI partition. -->

| Mount Point | Partition | Notes |
|---|---|---|
| /boot | Existing Windows EFI | Reuse, do not format |
| / | New Linux root | Created from freed space |

---

## After Installing Arch

<!-- How to make both OS entries appear in systemd-boot.
     Any Windows-specific gotchas after the install. -->

---

## Troubleshooting

**Windows won't boot after install**
>

**BitLocker recovery key prompt on boot**
>

**systemd-boot doesn't show Windows entry**
>

---

> 📖 **Further reading:** [Arch Wiki — Dual boot with Windows](https://wiki.archlinux.org/title/Dual_boot_with_Windows)
