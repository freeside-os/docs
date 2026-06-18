# Straylight Local Package Installation Engine

## Overview

The `straylight install-pkg` command provides a fast, direct local package installation pipeline that bypasses the remote `casync` chunking system. It is designed for local development loops where packages built via `straylight build` need to be installed directly into a staging system root.

```sh
straylight install-pkg <path_to_pkg.tar.gz>
```

## Architecture

### Target Isolation Model

To support stateless transitions and safe containerized bootstrapping, `install-pkg` **never writes to the host system root**. Instead, it targets two configurable paths:

| Environment Variable | Purpose | Default Fallback |
|:---|:---|:---|
| `STRAYLIGHT_RW_SYSTEM_ROOT` | Writable staging root (e.g., `/usr.next`) | `/tmp/straylight_staging_root` |
| `STRAYLIGHT_PKG_CACHE_ROOT` | Offline local package cache directory | `/var/cache/straylight/packages/` |

### Package Registration Isolation

Package registration (updating `packages.toml`) targets the `/etc/` directory **within the staging root**, not the host machine's `/etc/`. This prevents dynamic state mutations from affecting the running system:

```
Target: <STRAYLIGHT_RW_SYSTEM_ROOT>/etc/freeside/packages.toml
```

## Execution Phases

### Phase 1: Package Integrity Verification

1. Opens the `.tar.gz` package at the given path.
2. Extracts `meta/files.toml` and `meta/package.manifest` into memory.
3. Parses both files and validates their structure. Aborts on failure.

### Phase 2: Cache the Package (Arch-Style Cache)

1. Reads `STRAYLIGHT_PKG_CACHE_ROOT` (default: `/var/cache/straylight/packages/`).
2. Creates the cache directory if missing (`fs::create_dir_all`).
3. Copies the `.tar.gz` into the cache as `<name>-<version>-<build_id>.tar.gz`.

### Phase 3: Register Package in Staging Ledger

Adds or updates the package entry under `[packages.local]` in the staging root's `packages.toml`:

```toml
[packages.local.<pkg_name>]
path = "<cache_root>/<pkg_name>-<version>-<build_id>.tar.gz"
version = "<version>"
```

**Path calculation:**
```
<STRAYLIGHT_RW_SYSTEM_ROOT>/etc/freeside/packages.toml
```

### Phase 4: Differential File Copy Engine

For each file declared in `meta/files.toml`:

1. **Map destination:** `<STRAYLIGHT_RW_SYSTEM_ROOT>/<relative_path>` (strips `./` prefix).
2. **Diff check:** Compare existing file against declared state:
   - File size
   - SHA256 checksum
   - UNIX permissions (mode)
   - Owner (uid) and group (gid)
3. **Conditional extraction:**
   - **Identical:** Skip entirely (minimizes SSD/NVMe write wear).
   - **Different or missing:** Extract from tarball, write atomically (temp file + rename), then apply declared permissions and ownership.

### Phase 4b: Hook Execution

If the package contains scripts in `meta/hooks/`:
1. Extracts each hook to a temporary directory.
2. Executes hooks in sorted order with `STRAYLIGHT_RW_SYSTEM_ROOT` and package metadata exported as environment variables.
3. Cleans up temporary hook files after execution.

## Data Structures

### files.toml Schema (inside package tarball at `meta/files.toml`)

```toml
[[files]]
path = "./usr/bin/kitty"
sha256 = "a1b2c3d4..."
size = 1048576
mode = "0755"
uid = 0
gid = 0
```

### Rust Types

```rust
pub struct FileRecord {
    pub path: String,       // Relative path (e.g., "./usr/bin/kitty")
    pub sha256: String,     // Hex SHA256 hash
    pub size: u64,          // File size in bytes
    pub mode: String,       // Octal UNIX permissions (e.g., "0755")
    pub uid: u32,
    pub gid: u32,
}

pub struct FilesManifest {
    pub files: Vec<FileRecord>,
}
```

## Safety Guarantees

- **Atomic writes:** All file extractions use a write-to-temp-then-rename pattern to prevent corruption on interruption.
- **Directory creation:** Parent directories are created dynamically with `fs::create_dir_all` before file extraction.
- **Idempotent:** Running `install-pkg` multiple times with the same package is safe — identical files are skipped.
- **Non-destructive chown:** Ownership changes that fail (e.g., without root) produce warnings but do not abort the installation.

## Example Usage

```sh
# Build a package
sudo straylight build packages/kitty

# Install it to a staging root
STRAYLIGHT_RW_SYSTEM_ROOT=/usr.next \
STRAYLIGHT_PKG_CACHE_ROOT=/var/cache/straylight/packages \
straylight install-pkg build/packages/kitty-0.35.2-1.tar.gz
```

## CLI Output

The installer provides structured output with phase headers, per-file status indicators, and a final summary:

```
── Phase 4: Differential File Copy ──
  + created: usr/bin/kitty
  ~ updated: usr/lib/kitty/logo.png (sha256)
  (skipped 14 identical files)

  Summary: 1 created, 1 updated, 14 skipped (identical)
```
