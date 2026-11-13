# Customizing the Freeside OS Early Userspace (Initramfs & UKI)

Freeside OS utilizes a stateless, hermetic early boot process where systemd is loaded directly from a Unified Kernel Image (UKI) containing the kernel, command line parameters, and a compressed CPIO initramfs archive. 

This guide describes how package builders and developers can customize the initramfs, build a custom CPIO, and repackage it into a new UEFI bootable UKI.

---

## 1. Architectural Role of fstools

The `fstools` utility (specifically the `fstools initrd` subcommand group) is designed to generate early boot configurations programmatically without needing `root` or `sudo` privileges on the host. It:
1.  Loads binary packages from the local compile cache.
2.  Extracts files matching the runtime layout rules.
3.  Injects UEFI boot mount scripts.
4.  Creates character device files (e.g. `/dev/console` and `/dev/null`) programmatically.
5.  Compresses the final CPIO archive using gzip.

---

## 2. Customizing Initramfs Contents

The baseline packages included in the initramfs are defined in the `fstools` codebase (`fstools/src/fstools/initrd.py`).

To customize the packages included in the initramfs:
1.  **Add/Remove Packages:** Edit the `REQUIRED_BOOT_PACKAGES` list in `fstools/src/fstools/initrd.py` (which includes `musl`, `base-files`, `systemd`, `cryptsetup`, `btrfs-progs`, `util-linux`, `kmod`, `bash`, `libcap`, `openssl`, `xz`, `zstd`, `zlib`, `json-c`, and `lvm2`) to specify additional packages (e.g., adding `strace` or custom network tools).
2.  **Filter Payload Paths:** Adjust `PAYLOAD_EXCLUDE_FILTERS` in the same file to exclude files that are not critical for early booting, which keeps the initramfs lightweight and fast to load.
3.  **VFS Hierarchy:** Define additional directories or compatibility symlinks in `VFS_SKELETON_DIRECTORIES` or `VFS_COMPATIBILITY_SYMLINKS`.

---

## 3. Repackaging the Initramfs

Once your package dependencies are compiled, you can build a new CPIO archive directly using `fstools`:

```bash
fstools initrd build \
    --packages-dir build/packages \
    --output build/initramfs-custom.cpio.gz \
    --work-dir build/initramfs-custom-staging
```

### Scripted Verification
You can inspect the contents of the generated CPIO archive to verify all files and permissions are correct:

```bash
fstools initrd list build/initramfs-custom.cpio.gz
```

---

## 4. Re-assembling the Unified Kernel Image (UKI)

To construct a bootable UEFI PE executable (`.efi` file) from your customized initramfs:

1.  **Extract the Linux Kernel:**
    Retrieve the standard Freeside kernel from the compiled `linux-kernel` package cache:
    ```bash
    fstools pkg info linux-kernel
    # Look up the tarball location and extract:
    # usr/lib/freeside/linux/kernel/vmlinuz
    ```

2.  **Stitch the UKI:**
    Use `ukify` (provided by the `systemd` package group) on the host or inside a container to stitch the components together:
    ```bash
    ukify build \
        --linux=vmlinuz \
        --initrd=build/initramfs-custom.cpio.gz \
        --cmdline="console=ttyS0,115200 earlyprintk=serial,ttyS0,115200 systemd.volatile=overlay" \
        --output=build/EFI/Linux/freeside-custom.efi
    ```

3.  **Boot the Custom UKI:**
    Copy `freeside-custom.efi` directly to the EFI System Partition (ESP) at `/boot/EFI/Linux/` or boot it under QEMU using the live CD building target.

For more details on package manifest specs, refer to the [Package Packaging Guide](packaging_guide.md) or the [Live Image Build Guide](live_image_build.md).
