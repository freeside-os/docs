# Freeside Packaging Guide

This guide is for package maintainers describing how to write package manifests, write build recipes (`justfile`s), manage source files, and structure packages in the Freeside OS package repository.

---

## 1. Package Directory Structure

Every package lives in its own subdirectory under the global `packages/` directory:

```
packages/<package-name>/
├── package.manifest     # Package metadata, sources, and dependencies (TOML)
├── package.justfile     # Build and installation recipes (Justfile)
├── files/               # (Optional) Config templates, systemd units, etc.
└── patches/             # (Optional) Source patches
```

---

## 2. The Package Manifest (`package.manifest`)

The manifest is written in TOML and defines the package properties, compile-time/runtime dependencies, sources, and custom build environment variables.

### Example Manifest: `packages/openssl/package.manifest`
```toml
[package]
name = "openssl"
version = "3.3.0"
description = "Secure Sockets Layer toolkit"
dependencies = ["musl"]
group = "base"

[[sources]]
url = "https://www.openssl.org/source/openssl-3.3.0.tar.gz"
checksum = { algorithm = "sha256", value = "53e66b043322a606abf0087e7699a0e033a37fa13feb9742df35c3a33b18fb02" }

[build]
dependencies = ["musl"]

[build.environment]
CONFIGURE_ARGS = "--prefix=/usr --openssldir=/etc/ssl --libdir=lib shared zlib-dynamic"
```

### Manifest Sections

#### `[package]`
*   **`name`** (string): The unique name of the package.
*   **`version`** (string): The upstream version of the software.
*   **`description`** (string): A short description of the package.
*   **`dependencies`** (array of strings): **Runtime dependencies** needed by the packaged binaries.
*   **`group`** (string): The profile classification:
    *   `base`: Core runtime packages (libc, shell, ssl, basic tools).
    *   `builder`: Compilers, linkers, development libraries, and build systems.
    *   `system`: User-space system utilities (init systems, package manager, editors).

#### `[[sources]]` (array of tables)
Freeside supports multiple sources per package. They are downloaded or copied into the build workspace's `src/` directory.
*   **URL Source**: Downloads a remote archive.
    ```toml
    [[sources]]
    url = "https://ftp.gnu.org/gnu/bash/bash-5.2.21.tar.gz"
    checksum = { algorithm = "sha256", value = "c8e31bdc59b69aaffc5b36509905ba3e5cbb12747091d27b4b977f078560d5b8" }
    ```
*   **Local File Source**: Copies a file from the package directory itself (e.g. patches, configs).
    ```toml
    [[sources]]
    file = "etc.mount"
    checksum = { algorithm = "sha256", value = "c548b6f35a37d8df161fe4ae6b7323c9759d1615c27187677e09b02c8138b903" }
    ```
*   **Git Source**: Clones a Git repository.
    ```toml
    [[sources]]
    git = "https://github.com/example/project.git"
    ref = "v1.0.0"  # Can be a branch, tag, or commit hash
    ```

> [!IMPORTANT]
> All `url` and `file` sources require a `checksum` table containing the `algorithm` (currently `sha256`) and the expected hex `value`.

#### `[build]`
*   **`dependencies`** (array of strings): **Build-time dependencies** (compilers, build scripts, development headers) required to compile the package.

#### `[build.environment]` (or `[build.env]`)
Key-value pairs defining environment variables that will be exported to the shell when running build and package recipes. This is the correct place to specify custom compiler and linker flags (such as `CFLAGS` or `LDFLAGS`).
```toml
[build.environment]
CFLAGS = "-O3"
LDFLAGS = "-Wl,-O1"
CONFIGURE_ARGS = "--prefix=/usr --enable-shared"
MAKE_FLAGS = "-j8"
```

### Automatic Environment Variables

The packaging system automatically exports the following metadata variables to the build environment:

*   **`DESTDIR`**: The absolute path to the target staging directory (only exported during the `package` phase).
*   **`PKG_NAME`**: The name of the package.
*   **`PKG_VERSION`**: The version of the package.
*   **`PKG_DESCRIPTION`**: A short description of the package.
*   **`PKG_DEPENDENCIES`**: Space-separated list of runtime dependencies.
*   **`PKG_GROUP`**: The package group (e.g. `base`, `builder`).

---

## 3. The Build Recipe (`package.justfile`)

The build recipe is standard `justfile` syntax. It must implement two mandatory recipes: `build` and `package`.

```just
build:
    # 1. Unpack source archives if any
    tar -xf $PKG_NAME-$PKG_VERSION.tar.gz
    # 2. Configure, build, and compile the software
    # Environment variables from build.environment are automatically inherited!
    cd $PKG_NAME-$PKG_VERSION && ./configure $CONFIGURE_ARGS && make -j$(nproc)

package:
    # 3. Stage the files into the target output directory
    cd $PKG_NAME-$PKG_VERSION && make DESTDIR="$DESTDIR" install
    # 4. Create convenience symlinks, fix permissions, and perform adjustments
    ln -sf bash "$DESTDIR/usr/bin/sh"
    find "$DESTDIR" -type d -exec chmod 755 {} +
```

### Key Execution Rules
*   **Working Directory**: Straylight runs `just` with the working directory set to the package workspace. The extracted sources are placed in the `src/` subdirectory.
    *   To specify the source directory context, Straylight passes `-d src/` to `just`. This means your commands run relative to `src/`.
*   **Staging Directory (`DESTDIR`)**: The `package` recipe has access to the `DESTDIR` environment variable, which holds the absolute path to the staging folder. Your installation commands must write files *only* to this prefix (usually via `DESTDIR="$DESTDIR"` or `--prefix="$DESTDIR/usr"`).
*   **Permissions Cleanup**: Always ensure directories and binaries have standard permissions inside the staging directory (e.g. `chmod 755` for directories/executable binaries, `chmod 644` for libraries and headers).

---

## 4. How Straylight Compiles Packages

Understanding the compilation pipeline helps troubleshoot build issues:

1.  **Workspace Isolation**: Straylight creates an isolated workspace under `$STRAYLIGHT_BUILDER_ROOT/workspace/<package-name>-<version>/`.
2.  **Source Retrieval**:
    *   Downloads remote URLs, clones Git repositories, and copies local files declared in the manifest `sources` section.
    *   All retrieved sources are placed inside `src/`.
3.  **Justfile Staging**: Copies `package.justfile` to `package.justfile` in the workspace root.
4.  **Sandbox Spawning**: Spawns a `systemd-nspawn` container using `$STRAYLIGHT_BUILDER_ROOT/sandbox` as the rootfs. It bind-mounts the package workspace to `/workspace` inside the container.
5.  **Build Phase**: Runs `/usr/bin/just -f /workspace/package.justfile -d /workspace/src build` inside the container. All environment variables defined in the manifest's `[build.environment]` are exported to the container shell.
6.  **Package Phase**: Runs `/usr/bin/just -f /workspace/package.justfile -d /workspace/src package` inside the container with the `DESTDIR=/workspace/dest` environment variable set.
7.  **Packaging**: Straylight reads the files installed in `/workspace/dest/`, builds a `meta/files.toml` file list ledger, packages the directories into a compressed tarball named `<package-name>-<version>-1.tar.gz`, and saves it to `$STRAYLIGHT_BUILDER_ROOT/packages/`.

---

## 5. Installing Locally Built Packages

After building a package with `straylight build`, you can install it directly into a staging system root using the local package installer:

```sh
straylight install-pkg build/packages/<package-name>-<version>-1.tar.gz
```

This command:

1. **Verifies** the package integrity by parsing `meta/files.toml` and `meta/package.manifest` from the tarball.
2. **Caches** the package tarball to `STRAYLIGHT_PKG_CACHE_ROOT` (default: `/var/cache/straylight/packages/`).
3. **Registers** the package under `[packages.local]` in the staging root's `packages.toml`.
4. **Differentially copies** only changed or missing files into `STRAYLIGHT_RW_SYSTEM_ROOT`, minimizing disk writes.
5. **Executes hooks** from `meta/hooks/` if present.

### Environment Variables

| Variable | Purpose | Default |
|:---|:---|:---|
| `STRAYLIGHT_RW_SYSTEM_ROOT` | Target staging root for file installation | `/tmp/straylight_staging_root` |
| `STRAYLIGHT_PKG_CACHE_ROOT` | Package cache directory | `/var/cache/straylight/packages/` |

See [Local Package Installation Architecture](architecture/local_package_installation.md) for full technical details.

