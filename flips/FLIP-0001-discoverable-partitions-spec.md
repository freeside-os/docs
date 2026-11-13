---
flip: 0001
title: "Zero-Config Boot via systemd Discoverable Partitions Specification (DPS)"
authors: ["Lead Engineer & Freeside OS Core Team"]
status: accepted
created: 2026-07-28
type: architectural
component: installer
---

## Summary

Transition from explicit kernel command-line parameters (`rd.luks.name=...`, `root=/dev/mapper/...`, `rootflags=subvol=...`) to zero-config early boot using systemd's Discoverable Partitions Specification (`root=gpt-auto`).

## Motivation

- **Verbose Cmdline**: Machine-specific UUIDs and hardcoded mapper names (`freeside_crypt`) make bootloader configurations brittle.
- **Systemd Alignment**: The Discoverable Partitions Specification (DPS) is the standard across modern systemd distributions for automatic device discovery.
- **Clean UKI Support**: Enables identical, signed Unified Kernel Images (UKIs) without machine-specific parameters.

## Specification

### 1. Partition Type UUIDs
Assign exact DPS Partition Type UUIDs during disk partitioning:
- **Linux Root (x86-64)**: `4f68bce3-e8cd-4db1-96e7-fbcaf984b709`
- **EFI System Partition**: `c12a7328-f81f-11d2-ba4b-00a0c93ec93b`

### 2. Default Btrfs Subvolume
Designate `@usr_gen0` as the default Btrfs subvolume during filesystem creation:
```bash
btrfs subvolume set-default /mnt/freeside-install/@usr_gen0
```

### 3. Type #2 UKI Auto-Discovery
Deploy the UKI PE binary directly to `/boot/EFI/Linux/freeside.efi` (and `/boot/EFI/BOOT/BOOTX64.EFI` for UEFI fallback) with `root=gpt-auto` embedded in the UKI `.cmdline` section, requiring only a minimal `/boot/loader/loader.conf` (`timeout 3`).

## Implementation

1. **Storage Service**: `StorageService.set_root_partition_type` applies the DPS Linux Root (x86-64) GPT Partition Type UUID via `sfdisk --part-type`.
2. **Btrfs Service**: `BtrfsService.create_subvolumes` executes `btrfs subvolume set-default` on `@usr_gen0`.
3. **Bootloader Service**: `BootloaderService.deploy_uki` copies the UKI to `/boot/EFI/Linux/freeside.efi` and `/boot/EFI/BOOT/BOOTX64.EFI` and writes `/boot/loader/loader.conf`.
