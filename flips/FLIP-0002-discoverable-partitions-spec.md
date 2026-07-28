# FLIP-0002: Zero-Config Boot via systemd Discoverable Partitions Specification (DPS)

- **Status**: Proposed
- **Authors**: Lead Engineer & Freeside OS Core Team
- **Created**: 2026-07-28

---

## 1. Summary

This proposal outlines the architectural evolution from explicit kernel command-line parameters (`rd.luks.name=...`, `root=/dev/mapper/...`, `rootflags=subvol=...`) to zero-config early boot using systemd's [Discoverable Partitions Specification (DPS)](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/). 

By assigning uapi/systemd-standard GPT Partition Type UUIDs during installation and designating `@usr_gen0` as the default Btrfs subvolume, Freeside OS can boot installed systems cleanly using `options root=gpt-auto`.

---

## 2. Motivation

### Current State
Currently, `freeside-installer` generates bootloader entries (`freeside.conf`) containing explicit parameters:
```ini
title   Freeside OS
linux   /vmlinuz
initrd  /initramfs.cpio.gz
options rd.luks.name=UUID=freeside_crypt root=/dev/mapper/freeside_crypt rootflags=subvol=@usr_gen0 console=ttyS0,115200 earlyprintk=serial,ttyS0,115200
```

### Problems & Disadvantages
1. **Verbose Kernel Cmdline**: Command line strings are brittle, tied to specific partition UUIDs, and mandate hardcoded mapper names (`freeside_crypt`).
2. **Non-Standard Partitioning**: Generic partitioning tools often assign generic Linux filesystem GUIDs (`0fc63daf-8483-4772-8e79-3d69d8477de4`), preventing systemd from automatically discovering encrypted LUKS root partitions.
3. **Btrfs Subvolume Explicit Flag**: Requiring `rootflags=subvol=@usr_gen0` on the kernel command line forces initramfs generators to handle custom flags instead of delegating default root selection to the Btrfs filesystem layer.

---

## 3. Detailed Specification

### A. Partition Type UUID Assignment
During disk partitioning, `StorageService` in `freeside-installer` will set exact DPS Partition Type UUIDs:

| Partition Role | Standard Type Name | Partition Type UUID |
| :--- | :--- | :--- |
| **EFI System Partition (ESP)** | EFI System Partition | `c12a7328-f81f-11d2-ba4b-00a0c93ec93b` |
| **Encrypted Root Partition** | Root Partition (x86-64 LUKS) | `7ff06982-63b1-4b5f-927a-01a257c2c7de` |
| **Generic Root Partition (Unencrypted)** | Root Partition (x86-64) | `4f68b4e7-0396-49d6-bf10-2e693739070a` |

### B. Btrfs Default Subvolume Registration
During the Btrfs subvolume setup step (`BtrfsService`), the installer will mark `@usr_gen0` as the default subvolume for the filesystem pool:
```bash
btrfs subvolume set-default <subvol_id_of_@usr_gen0> /mnt/freeside-install
```
When `mount` is invoked without explicit `subvol=` options, Btrfs automatically mounts `@usr_gen0` as the root filesystem.

### C. Bootloader Configuration Simplification
The generated `loader/entries/freeside.conf` simplifies to:
```ini
title   Freeside OS
linux   /vmlinuz
initrd  /initramfs.cpio.gz
options root=gpt-auto console=ttyS0,115200 earlyprintk=serial,ttyS0,115200
```

---

## 4. Implementation Plan

1. **Installer Storage Update**:
   - Update `StorageService` in [installer/src/freeside_installer/services/storage.py](installer/src/freeside_installer/services/storage.py) to pass DPS Type UUIDs (`7ff06982-63b1-4b5f-927a-01a257c2c7de`) to `sfdisk` / `parted` when partitioning.
2. **Installer Btrfs Update**:
   - Update `BtrfsService.create_subvolumes()` in [installer/src/freeside_installer/services/btrfs.py](installer/src/freeside_installer/services/btrfs.py) to invoke `btrfs subvolume set-default` targeting `@usr_gen0`.
3. **Installer Bootloader Update**:
   - Update `BootloaderService.deploy_kernel_initramfs()` in [installer/src/freeside_installer/services/bootloader.py](installer/src/freeside_installer/services/bootloader.py) to use `options root=gpt-auto console=ttyS0,115200 earlyprintk=serial,ttyS0,115200`.

---

## 5. Rationale & Alternatives

- **Alternative (Maintain explicit options)**: Keeps current setup working, but requires maintaining custom cmdline option strings and prevents zero-config UEFI UKI setups.
- **Why DPS**: DPS is standard across modern systemd distributions (Arch, Fedora, openSUSE). It integrates natively with `systemd-gpt-auto-generator`, `systemd-cryptsetup`, and UKI signing architectures.

---

## 6. Security & Maintainability Implications

- **Security**: Aligns cleanly with TPM2 unsealing (`systemd-cryptenroll`) and Secure Boot, as the command-line parameters inside the UKI binary remain constant across kernel versions.
- **Maintainability**: Removes machine-specific partition UUIDs from kernel config files, enabling byte-identical boot entries across deployments.
