# Freeside OS: Live Image Build & Boot Guide

This guide explains the live hybrid ISO architecture of **Freeside OS**, how to build the SquashFS live system image, and how to run it in a virtualized QEMU environment.

---

## 1. Architectural Overview

Freeside OS uses a stateless, read-only live architecture. The system boots from a hybrid ISO and mounts a compressed SquashFS root filesystem using OverlayFS for dynamic read-write access in memory.

### Boot Sequence

```mermaid
graph TD
    A[BIOS/UEFI] -->|Loads ESP| B[systemd-boot]
    B -->|Loads Kernel & Initramfs| C[Linux Kernel]
    C -->|Launches| D[Initramfs systemd]
    D -->|Executes Mount Units| E[run-initramfs-live.mount]
    E -->|Mounts ISO CDROM| F[Loop Mount rootfs.squashfs]
    F -->|Mounts SquashFS RO| G[/sysroot]
    G -->|Overlays tmpfs RW| H[OverlayFS /]
    H -->|Switch Root| I[Stateless Active OS]
```

1. **UEFI / systemd-boot**: The bootloader loads the Linux kernel (`vmlinuz`) and the initramfs (`initrd`) from the ISO's EFI System Partition (ESP).
2. **Initramfs systemd**: systemd is initialized inside the initramfs. It parses the kernel cmdline (which contains no hardcoded `root=` option, utilizing systemd's partition auto-discovery instead).
3. **Mounting the ISO**: The custom mount unit `run-initramfs-live.mount` automatically locates the boot medium (by scanning disk labels for `FREESIDE_LIVE` or detecting the optical drive) and mounts it to `/run/initramfs/live/`.
4. **Loop Mount SquashFS**: The custom mount unit `sysroot.mount` loop-mounts the compressed root filesystem `/run/initramfs/live/rootfs.squashfs` to the target system root `/sysroot/`.
5. **OverlayFS Staging**: A tmpfs overlay is mounted on top of the read-only SquashFS `/sysroot` to allow temporary runtime mutations (configurations, logs, etc.).
6. **Switch Root**: systemd pivots/switches root from the initramfs to the `/sysroot` overlay, handing control over to the stateless target user space.

---

## 2. Prerequisites & Setup

### Host Build Dependencies

The ISO assembly runs on the host (not inside the Straylight sandbox) and requires the following tools:

| Tool | Arch Package | Purpose |
|---|---|---|
| `mkfs.vfat` | `dosfstools` | Creates the FAT filesystem for the EFI boot image |
| `mmd`, `mcopy` | `mtools` | Populates the FAT image without loopback mounts |
| `mksquashfs` | `squashfs-tools` | Compresses the root filesystem into SquashFS |
| `xorriso` | `libisoburn` | Compiles the final ISO 9660 image with UEFI boot |

### Host Runtime Dependencies

| Tool | Arch Package | Purpose |
|---|---|---|
| `qemu-system-x86_64` | `qemu-full` | UEFI virtual machine for booting the ISO |
| OVMF firmware | `edk2-ovmf` | UEFI firmware image (typically at `/usr/share/ovmf/x64/OVMF.4m.fd`) |

Install all dependencies on Arch Linux:
```bash
sudo pacman -S dosfstools mtools squashfs-tools libisoburn qemu-full edk2-ovmf
```

---

## 3. How to Build the Live Image

Building the live image requires compiling all packages in the `base`, `system`, and `linux` groups so that their compiled tarballs are cached, then assembling the final bootable ISO.

### Step 1: Ensure Sandbox Environment
Ensure that the Musl sandboxed container environment is bootstrapped:
```bash
just bootstrap::build-sandbox
```

### Step 2: Compile Package Groups
Compile the `base`, `system`, and `linux` package groups to populate the local tarball cache (`build/packages/`):
```bash
just pkg::build-group base
just pkg::build-group system
just pkg::build-group linux
```

### Step 3: Build the Live ISO
Assemble the final bootable ISO:
```bash
just live::build
```

This command:
1. Compiles any missing `live-image` tool dependencies (`mtools`, `xorriso`, `squashfs-tools`) inside the sandbox, so the host has the required ISO assembly utilities.
2. Runs `packages/fsimage.py` on the host, which:
   - Resolves the topologically-sorted package list for the `base` and `system` groups.
   - Extracts each package's compiled tarball into a SquashFS staging root (with UsrMerge symlinks).
   - Places the bootstrap environment (`sandbox-root.tgz`) and package tarballs inside `/var/cache/freeside/` within the SquashFS tree.
   - Compresses the root tree into `rootfs.squashfs` using `mksquashfs` with Zstd compression.
   - Assembles a 70 MB UEFI FAT boot image with systemd-boot, the kernel, and the initramfs.
   - Invokes `xorriso` with UEFI parameters and `-iso-level 3` (to allow files larger than 4 GiB) to output the hybrid ISO at `build/freeside-live.iso`.

> **Note:** The SquashFS rootfs is always built fresh from package tarballs; caching is disabled to prevent stale package configurations or systemd unit changes from being silently bypassed.

---

## 4. Running the Live Image

### Running via QEMU
Boot the live ISO directly with the provided just target:
```bash
just live::run
```

This runs QEMU with KVM acceleration, 2 GB of memory, OVMF UEFI firmware, and the ISO attached as a SATA/AHCI hard disk (required for DMA-based loading of the large initramfs). The serial console is redirected to the terminal:
```bash
qemu-system-x86_64 \
    -enable-kvm -cpu host -m 2G \
    -bios /usr/share/ovmf/x64/OVMF.4m.fd \
    -device ahci,id=sata \
    -drive file=build/freeside-live.iso,id=livecd,format=raw,if=none \
    -device ide-hd,bus=sata.0,drive=livecd \
    -nographic -serial mon:stdio
```

> **Note:** The ISO is attached as a SATA hard disk (`ide-hd`) rather than an IDE CD-ROM (`-cdrom`). This is required because the AHCI driver (`CONFIG_SATA_AHCI=y`) is built into the kernel, while the IDE/ATAPI CD-ROM drivers are compiled as modules not present in the initramfs. SATA also enables DMA for significantly faster boot file loading.

To exit QEMU, use the console shortcut: `Ctrl-A` then press `x`.
