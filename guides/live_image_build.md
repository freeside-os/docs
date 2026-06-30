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

Ensure you have the following host dependencies installed:
*   `qemu-system-x86_64` (for booting the virtual machine)
*   `OVMF` (Open Virtual Machine Firmware for UEFI boot, typically located at `/usr/share/ovmf/x64/OVMF.4m.fd` on Linux hosts)

---

## 3. How to Build the Live Image

Building the live image requires compiling all packages in the `base`, `builder`, and `system` groups so that their compiled tarballs are cached, then compiling the final `live-image` bootable package group.

### Step 1: Ensure Sandbox Environment
Ensure that the Musl sandboxed container environment is bootstrapped:
```bash
just bootstrap::build-sandbox
```

### Step 2: Compile All Package Groups
Compile all package groups to populate the local cache (`build/packages/`):
```bash
just build-package-group base
just build-package-group builder
just build-package-group system
```

### Step 3: Build the Live Image Group
Compile the final bootable live ISO package group:
```bash
just build-package-group live-image
```
This process:
1. Resolves package dependency lists.
2. Unpacks all cached package tarballs to a temporary SquashFS staging root.
3. Places the bootstrap environment (`sandbox-root.tgz`) and package tarballs inside `/var/cache/freeside/` within the SquashFS tree.
4. Compresses the root tree into a single `rootfs.squashfs` file using `mksquashfs` and Zstd compression.
5. Invokes `xorriso` with UEFI parameters and `-iso-level 3` (to allow files larger than 4 GiB) to output the hybrid ISO.

---

## 4. Extracting and Running the Live Image

### Extracting the ISO
Once compiled, extract the final ISO from the `live-image` package tarball to the host `build/` directory:
```bash
tar -xzf build/packages/live-image-1.0.0-1.tar.gz -C build/ ./freeside-live.iso
```

### Running via QEMU
Run the live system in QEMU using the provided helper script:
```bash
./run_iso.sh
```

The script runs QEMU with KVM acceleration, allocates 2 GB of memory, loads the OVMF UEFI firmware, mounts the generated live ISO, and redirects the console output to the terminal:
```bash
#!/usr/bin/env bash
qemu-system-x86_64 \
    -enable-kvm -cpu host -m 2G \
    -bios /usr/share/ovmf/x64/OVMF.4m.fd \
    -cdrom build/freeside-live.iso \
    -nographic -serial mon:stdio
```

To exit QEMU, use the console shortcut: `Ctrl-A` then press `x`.
