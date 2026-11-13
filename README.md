# Freeside OS Documentation Index

Welcome to the central documentation repository for **Freeside OS**—a next-generation, independent Linux distribution engineered for resilience, absolute predictability, and zero-maintenance overhead.

This repository (`docs`) is the single source of truth for the system's architectural blueprints, development workflows, packaging guidelines, and data contract schemas.

---

## 1. Project Repository Index

Freeside OS is structured using a multi-repository workspace layout:

*   **Workspace (`freeside`)**: The parent workspace root (`git@github.com:freeside-os/freeside.git`) coordinates development, `just` orchestration modules, and mounts sub-repositories.
*   **[bootstrap](https://github.com/freeside-os/bootstrap)**: Stage 0 and Stage 1 bootstrap engine and sandboxing system.
    *   *Remote*: `git@github.com:freeside-os/bootstrap.git`
*   **[packages](https://github.com/freeside-os/packages)**: Declarative package recipes (`package.manifest` & `package.justfile`) and build scripts.
    *   *Remote*: `git@github.com:freeside-os/packages.git`
*   **[straylight](https://github.com/freeside-os/straylight)**: Custom package manager CLI and system daemon (`straylightd`), written in Rust.
    *   *Remote*: `git@github.com:freeside-os/straylight.git`
*   **[fstools](https://github.com/freeside-os/fstools)**: Developer and builder tools CLI (`pkg`, `initrd`, `image`, `sandbox`), written in Python and packaged as a PEX zipapp.
    *   *Remote*: `git@github.com:freeside-os/fstools.git`
*   **[installer](https://github.com/freeside-os/installer)**: Declarative interactive TUI system installer (`freeside-install`), written in Python (`textual`) and packaged as a PEX zipapp.
    *   *Remote*: `git@github.com:freeside-os/installer.git`
*   **[wintermute](https://github.com/freeside-os/wintermute)**: Autonomous agentic development platform and packaging harness for Freeside OS.
    *   *Remote*: `git@github.com:freeside-os/wintermute.git`
*   **[docs](https://github.com/freeside-os/docs)**: System design specifications, guides, reference schemas, and FLIPs (this repository).
    *   *Remote*: `git@github.com:freeside-os/docs.git`

---

## 2. Documentation Directory Structure

The documentation is organized logically into **Specifications** (architectural rules), **Guides** (step-by-step developer manuals), **Reference** (schemas and profile catalogs), **FLIPs** (Freeside Language & Enhancement Proposals), and **Ideation**.

### A. Specifications
*   **[System Architecture](specifications/system_architecture.md)**: Details the filesystem layout, Btrfs subvolumes, OverlayFS overlays, UKI boot protocol, and the transactional Btrfs update and rollback mechanics.
*   **[Package Management & Deployment](specifications/package_management.md)**: Specifies the Straylight CLI/Daemon model, Unix socket peer authorization, differential package installation (`straylight install-pkg`), the ingestion pipeline, and the Wintermute curator test loop.
*   **[Bootstrapping & Sandbox](specifications/bootstrapping_sandbox.md)**: Outlines Stage 0/1 compilation phases, topological sorting of packages inside Docker Alpine, host rootfs assembly (`fstools sandbox build`), and Straylight container execution.
*   **[Simstim Developer Sandboxes](specifications/simstim_architecture.md)**: Specifies the declarative developer workstation subsystem, directory-based configuration (`.simstim`), Btrfs-backed snapshot lifecycle phases, host-guest UID/GID mapping, PTY terminal multiplexing, graphics/GPU forwarding, and nested rootless container virtualization.
*   **[Wintermute Architecture](specifications/wintermute_architecture.md)**: Specifies the multi-domain agentic engine, `ExecutionSpec`, `DomainRegistry`, and Kubernetes-style declarative workflow manifests.

### B. Guides
*   **[Package Packaging Guide](guides/packaging_guide.md)**: A developer's manual on how to write package metadata manifests (`package.manifest` in TOML) and build recipes (`package.justfile` using Just).
*   **[Local Development Guide](guides/local_development.md)**: Explains workspace checkouts, compiling the bootstrap sandbox, building `straylight`, `fstools`, and `installer`, building packages locally, running subproject test suites, and booting in QEMU.
*   **[Live Image Build & Boot Guide](guides/live_image_build.md)**: Details building the hybrid SquashFS Live ISO via `fstools image build` (`just live::build`) and booting/installing in QEMU (`just live::run`, `just live::boot-system`, `just live::update-uki`).
*   **[Customizing Early Userspace (Initramfs & UKI)](guides/customizing_initrd.md)**: Explains how `fstools initrd` constructs hermetic CPIO archives and how to customize early boot packages and UKIs.
*   **[Wintermute Platform Guide](guides/wintermute.md)**: Operational guide for running Wintermute CLI workflows and declarative YAML manifests.

### C. Reference
*   **[System Profiles & Package Registry](reference/profiles_registry.md)**: Maps profile inheritance (`bootstrap` -> `builder` -> `server` -> `desktop`) and catalog lists of packages per group.
*   **[Configuration & Metadata Schemas](reference/file_schemas.md)**: Standard schemas for manifests (`package.manifest`, `packages.toml`, `tree.toml`, `channel.toml`, `trees/<tree_hash>.toml`, and `files.toml`).

### D. FLIPs & Ideation
*   **[FLIP-0000: Proposal Template](flips/FLIP-0000-template.md)**: Standard template for Freeside enhancement proposals.
*   **[FLIP-0001: Discoverable Partitions Specification (DPS)](flips/FLIP-0001-discoverable-partitions-spec.md)**: Zero-config early boot using systemd GPT auto-discovery (`root=gpt-auto`) and `@usr_gen0` default Btrfs subvolumes.
*   **[Wintermute Platform Vision](ideation/wintermute_platform_vision.md)**: Long-term architectural vision for Wintermute as an autonomous OS engineering harness.
