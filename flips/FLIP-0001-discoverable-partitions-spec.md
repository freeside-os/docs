---
flip: 0001
title: "Zero-Config Boot via systemd Discoverable Partitions Specification (DPS)"
authors: ["Lead Engineer & Freeside OS Core Team"]
status: proposed
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
- **Root LUKS (x86-64)**: `7ff06982-63b1-4b5f-927a-01a257c2c7de`
- **EFI System Partition**: `c12a7328-f81f-11d2-ba4b-00a0c93ec93b`

### 2. Default Btrfs Subvolume
Designate `@usr_gen0` as the default Btrfs subvolume during filesystem creation:
```bash
btrfs subvolume set-default <subvol_id_of_@usr_gen0> /mnt/freeside-install
```

### 3. Simplified Bootloader Options
The loader entry `freeside.conf` options simplify to:
```ini
options root=gpt-auto console=ttyS0,115200 earlyprintk=serial,ttyS0,115200
```

## Implementation

1. **Storage Service**: Update `StorageService` to apply DPS Partition Type UUIDs.
2. **Btrfs Service**: Update `BtrfsService` to execute `btrfs subvolume set-default`.
3. **Bootloader Service**: Update `BootloaderService` to output `options root=gpt-auto`.
