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

Freeside OS bootstrapping is a two-stage process:

1.  **Stage 0: Alpine Host Sandbox (One-time Setup)**
    To build the initial musl-based compiler sandbox core inside a Docker container, execute:
    ```bash
    just bootstrap::build-sandbox
    ```
    This command runs Stage 0 Alpine host container assembly and packages the results to `build/bootstrap/sandbox-root.tgz`.

2.  **Stage 0 to Stage 1 Promotion (Manual Step)**
    Since Straylight expects the sandbox core at the root of the builder output, manually copy the sandbox tarball:
    ```bash
    cp build/bootstrap/sandbox-root.tgz build/sandbox-root.tgz
    ```

3.  **Stage 1: Pure Sandbox Recompilation**
    To rebuild the base and builder packages inside the isolated Freeside sandbox (discarding Alpine host dependencies and compiling pure Freeside packages):
    ```bash
    just sys::build-sandbox   # Or use alias: just build-sandbox
    ```
    This compiles the packages inside the sandbox and generates the final, pure `build/sandbox-root.tgz`.

### B. Compiling the Straylight CLI

To compile the `straylight` CLI manager:
1.  Execute the straylight build command:
    ```bash
    just straylight::build    # Or use alias: just build-straylight
    ```
2.  This builds the Rust binary under `straylight/` and copies the compiled executable output to `build/straylight`.

### C. Compiling Packages Locally

You can use the local `straylight` binary to build packages. The CLI requires three environment variables to define its locations:

*   `STRAYLIGHT_PACKAGES_ROOT`: Location of the package source directory (e.g. `$(pwd)/packages`).
*   `STRAYLIGHT_BUILDER_ROOT`: Location of the build sandbox and staging workspace (e.g. `$(pwd)/build`).
*   `STRAYLIGHT_BUILDER_OUTPUT_ROOT`: Location where compiled tarballs are written (e.g. `$(pwd)/build/packages`).

1.  Run the build command using `sudo -E` to preserve these variables:
    ```bash
    STRAYLIGHT_PACKAGES_ROOT="$(pwd)/packages" \
    STRAYLIGHT_BUILDER_ROOT="$(pwd)/build" \
    STRAYLIGHT_BUILDER_OUTPUT_ROOT="$(pwd)/build/packages" \
    sudo -E build/straylight build --pkg <package-name>
    ```
2.  **What happens under the hood:**
    *   Straylight checks for an extracted sandbox under `STRAYLIGHT_BUILDER_ROOT/sandbox/`.
    *   If missing, it locates `sandbox-root.tgz` and extracts it.
    *   It mounts the sandbox container, binds the package source folder and builder workspaces, and executes the compilation targets.
    *   Completed package outputs are compiled and written directly into `STRAYLIGHT_BUILDER_OUTPUT_ROOT`.


---

## 3. Maintenance & Clean Operations

To clean temporary workspaces, Cargo build outputs, and transient build sandboxes:
```bash
just sys::clean           # Or use alias: just clean
```

To purge compiled user-space packages stored under `build/packages/`:
```bash
just pkg::clean           # Or use alias: just clean-packages
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
