# Freeside OS: Configuration & Metadata Schemas

This document serves as the single source of truth for file formats, manifests, and structured configurations used across the Freeside OS ecosystem.

---

## 1. Package Manifest (`package.manifest`)

Stored under `packages/<name>/package.manifest` to declare metadata, compile/runtime dependencies, source locations, and verification hashes.

```toml
[package]
name = "pkg-name"                        # string: unique identifier
version = "1.0.0"                        # string: package version
description = "Short summary"            # string: descriptive summary
group = "base"                           # string: 'base' | 'builder' | 'system'
dependencies = ["musl"]                  # array of strings: runtime dependencies

[[sources]]
url = "https://example.com/src.tar.gz"   # string: remote source URL (exclusive with file and git)
checksum = { algorithm = "sha256", value = "<hex>" }  # table: verification checksum

# Alternative Local File Source:
# file = "config.patch"
# checksum = { algorithm = "sha256", value = "<hex>" }

# Alternative Git Source:
# git = "https://github.com/example/repo.git"
# ref = "v1.0.0"

[build]
dependencies = ["llvm", "make"]          # array of strings: build-time dependencies

[build.environment]
CFLAGS = "-O3"                           # key-value pairs: custom build variables
CONFIGURE_ARGS = "--prefix=/usr"
```

---

## 2. Package Build Recipe (`package.justfile`)

Stored under `packages/<name>/package.justfile` to define step-by-step commands to compile and package source codes.

```just
# package.justfile template

# Invoked to compile binary output
build:
    cd src-* && ./configure && make

# Invoked to stage binary output
package:
    cd src-* && make DESTDIR="$DESTDIR" install
```

---

## 3. Host Desired Target State (`/etc/freeside/packages.toml`)

Stored on client nodes to declare the target profile, active tree hash, and any localized packages.

```toml
[system]
profile = "desktop"                      # string: baseline system profile
tree = "<tree_hash>"                     # string: expected target tree commit hash

[packages]
apps = [                                 # array of strings: additive remote packages
    "kitty",
    "firefox"
]

[packages.local.zlib]                    # table: local package staging overrides
path = "/var/cache/straylight/packages/zlib-1.3.1.tar.gz"
version = "1.3.1"
```

---

## 4. Host Active State Ledger (`/etc/freeside/tree.toml`)

Generated automatically by `straylightd` after a successful synchronization. Read-only (`0444`).

```toml
tree = "<tree_hash>"                     # string: currently active tree hash
synced_at = 2026-06-18T10:44:47Z         # datetime: synchronization timestamp

[packages]                               # table: packages present on the system
musl = "1.2.5-1"
uutils = "0.0.28-1"
systemd = "260.2"
```

---

## 5. Remote Distribution Channel State (`channel.toml`)

Maintained on upstream CDN endpoints to indicate the latest stable release targets.

```toml
channel = "stable"                       # string: channel identifier ('stable' | 'beta')
updated_at = 2026-06-18T12:00:00Z        # datetime: upstream update timestamp
latest_tree = "<tree_hash>"              # string: latest stable tree hash
latest_uki_sha256 = "<sha256_hex>"       # string: checksum of Unified Kernel Image
```

---

## 6. Upstream Tree Manifest (`trees/<tree_hash>.toml`)

Maintained on upstream mirrors. Maps release-wide packages to their content-addressable indices.

```toml
tree = "<tree_hash>"                     # string: tree hash identifier

[profiles.desktop]                       # table: profile package compositions
packages = ["musl", "systemd", "mesa"]

[packages.musl]                          # table: package index routes
version = "1.2.5-1"
caidx = "packages/musl-1.2.5-1.caidx"
```

---

## 7. Package Files Listing Schema (`meta/files.toml`)

Included inside compiled package tarballs at `meta/files.toml` to list contents and permission hashes.

```toml
[[files]]
path = "./usr/bin/kitty"                 # string: relative target path
sha256 = "<sha256_hex>"                  # string: file verification hash
size = 1048576                           # integer: size in bytes
mode = "0755"                            # string: UNIX file mode
uid = 0                                  # integer: owner UID
gid = 0                                  # integer: owner GID
```
