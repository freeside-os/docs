# Freeside OS: Local Development Guide

This guide describes how to set up your local development workspace, build the bootstrapping compiler sandbox, develop package recipes, compile them locally, and test the operating system states.

---

## 1. Developer Workspace Setup

Freeside is structured as multiple repositories checked out under a single parent workspace folder. To set up your local workspace, check out the repositories into the following directory layout:

```text
freeside/ (Workspace Root)
├── bootstrap/                   # git@github.com:freeside-os/bootstrap.git
├── packages/                    # git@github.com:freeside-os/packages.git
├── straylight/                  # git@github.com:freeside-os/straylight.git
└── docs/                        # git@github.com:freeside-os/docs.git
```

### Host Requirements

Your local development machine must satisfy the following:
*   **Operating System:** Linux (Arch Linux is recommended)
*   **Privileges:** Sudo/root access (required for container sandboxing via `systemd-nspawn`)
*   **Required Packages:** `just` (>= 1.51.0), `systemd` (with `systemd-nspawn`), `curl`, `tar`, `docker` (optional, for Stage 0 build)

---

## 2. Compilation Workflows

### A. Building the Bootstrapping Sandbox

To build the self-contained rootfs environment and stage compiler binaries:
1.  From the workspace root, execute the bootstrap target:
    ```bash
    just build-bootstrap
    ```
2.  This command runs Stage 0 Alpine host container assembly and stages the results.
3.  Once finished, the output is saved to `build/artifacts/freeside-builder-core.tar.gz`.

### B. Compiling the Straylight CLI

To compile the `straylight` CLI manager:
1.  Execute the build-straylight command:
    ```bash
    just build-straylight
    ```
2.  This builds the Rust binary under `straylight/` and copies the compiled executable output to `build/straylight`.

### C. Compiling Packages Locally

You can use the local `straylight` binary to build any package declared in `packages/`:
1.  Run the build subcommand using sudo (required for chroot/nspawn containment):
    ```bash
    sudo build/straylight build packages/<package-name>
    ```
2.  **What happens under the hood:**
    *   Straylight checks for an extracted sandbox under `build/sandbox/`.
    *   If missing, it locates `build/artifacts/freeside-builder-core.tar.gz` and extracts it.
    *   It mounts the sandbox container, binds the package source folder, and executes the compilation targets.
    *   Completed package outputs are compiled into `build/packages/`.

### D. Single-Step Release Build

To run the compile, package, and distribution index ingestion in a single stride:
```bash
sudo build/straylight release packages/<package-name>
```

---

## 3. Maintenance & Clean Operations

To clean temporary workspaces, Cargo build outputs, and transient build sandboxes:
```bash
just clean
```

To purge compiled user-space packages stored under `build/packages/`:
```bash
just clean-packages
```

---

## 4. Boot Emulation & VM Verification

To simulate boot-loading and systemd target activation before deployment:
1.  Ensure you have `qemu-system-x86_64` and `swtpm` installed on your development machine.
2.  Run the boot emulator suite:
    ```bash
    # (Optional) Invokes QEMU micro-VM loading candidate UKI
    just test-boot
    ```
3.  This validates that OverlayFS mounts correctly, the `/usr` subvolume is read-only, and systemd reaches `multi-user.target` with zero failures.
