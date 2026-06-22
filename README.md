# Freeside OS Documentation Index

Welcome to the central documentation repository for **Freeside OS**—a next-generation, independent Linux distribution engineered for resilience, absolute predictability, and zero-maintenance overhead.

This repository (`docs`) is the single source of truth for the system's architectural blueprints, development workflows, packaging guidelines, and data contract schemas.

---

## 1. Project Repository Index

Freeside OS is structured using a multi-repository workspace layout:

*   **Workspace (freeside)**: The parent workspace root coordinates development and mounts sub-repositories.
*   **[bootstrap](git@github.com:freeside-os/bootstrap.git)**: Stage 0 and Stage 1 bootstrap engine and sandboxing system.
    *   *Remote*: `git@github.com:freeside-os/bootstrap.git`
*   **[packages](git@github.com:freeside-os/packages.git)**: Declarative package recipes and build scripts.
    *   *Remote*: `git@github.com:freeside-os/packages.git`
*   **[straylight](git@github.com:freeside-os/straylight.git)**: Custom package manager CLI and system daemon (`straylightd`), written in Rust.
    *   *Remote*: `git@github.com:freeside-os/straylight.git`
*   **[docs](git@github.com:freeside-os/docs.git)**: System design specifications, guides, and diagrams (this repository).
    *   *Remote*: `git@github.com:freeside-os/docs.git`

---

## 2. Documentation Directory Structure

The documentation is organized logically into **Specifications** (architectural rules), **Guides** (step-by-step developer manuals), and **Reference** (schemas and profile catalogs).

### A. Specifications
*   **[System Architecture](file:///home/dq/Code/freeside/docs/specifications/system_architecture.md)**: Details the filesystem layout, Btrfs subvolumes, OverlayFS overlays, UKI boot protocol, and the transactional Btrfs update and rollback mechanics.
*   **[Package Management & Deployment](file:///home/dq/Code/freeside/docs/specifications/package_management.md)**: Specifies the Straylight CLI/Daemon model, Unix socket peer authorization, differential package installation (`straylight install-pkg`), the ingestion pipeline, and the Wintermute curator test loop.
*   **[Bootstrapping & Sandbox](file:///home/dq/Code/freeside/docs/specifications/bootstrapping_sandbox.md)**: Outlines Stage 0/1 compilation phases, topological sorting of packages inside Docker Alpine, host rootfs assembly, and Straylight container chroot execution.
*   **[Simstim Developer Sandboxes](file:///home/dq/Code/freeside/docs/specifications/simstim_architecture.md)**: Specifies the declarative developer workstation subsystem, directory-based configuration (.simstim), Btrfs-backed snapshot lifecycle phases, host-guest UID/GID mapping, PTY terminal multiplexing, graphics/GPU forwarding, and nested rootless container virtualization.

### B. Guides
*   **[Package Packaging Guide](file:///home/dq/Code/freeside/docs/guides/packaging_guide.md)**: A developer's manual on how to write package metadata manifests (`package.manifest` in TOML) and build recipes (`package.justfile` using Just).
*   **[Local Development Guide](file:///home/dq/Code/freeside/docs/guides/local_development.md)**: Explains workspace checkouts, compiling the bootstrap sandbox, building the Straylight CLI, building packages locally, and running boot emulation in QEMU micro-VMs.

### C. Reference
*   **[System Profiles & Package Registry](file:///home/dq/Code/freeside/docs/reference/profiles_registry.md)**: Maps profile inheritance (bootstrap $\rightarrow$ builder $\rightarrow$ server $\rightarrow$ desktop) and catalog lists of packages per profile.
*   **[Configuration & Metadata Schemas](file:///home/dq/Code/freeside/docs/reference/file_schemas.md)**: Standard schemas for manifests (`package.manifest`, `packages.toml`, `tree.toml`, `channel.toml`, `trees/<tree_hash>.toml`, and `files.toml`).
