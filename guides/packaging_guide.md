# Freeside OS: Package Packaging Guide

This guide is for package maintainers describing how to write package manifests, build recipes (`justfile`s), manage source files, and structure packages in the Freeside OS package repository.

---

## 1. Package Directory Structure

Every package lives in its own subdirectory under the global `packages/` directory:

```text
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

*   **URL Source:** Downloads a remote archive.
    ```toml
    [[sources]]
    url = "https://ftp.gnu.org/gnu/bash/bash-5.2.21.tar.gz"
    checksum = { algorithm = "sha256", value = "c8e31bdc59b69aaffc5b36509905ba3e5cbb12747091d27b4b977f078560d5b8" }
    ```
*   **Local File Source:** Copies a file from the package directory itself (e.g. patches, configs).
    ```toml
    [[sources]]
    file = "etc.mount"
    checksum = { algorithm = "sha256", value = "c548b6f35a37d8df161fe4ae6b7323c9759d1615c27187677e09b02c8138b903" }
    ```
*   **Git Source:** Clones a Git repository.
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

---

## 3. The Build Recipe (`package.justfile`)

The build recipe is written in the `just` execution syntax and defines how to compile the source and stage its files.

### Standard targets:

*   **`build`**: Compiles the source files in the container workspace.
*   **`package`**: Installs files into the staging environment prefix designated by `$DESTDIR`.

### Example Build Recipe: `packages/zlib/package.justfile`

```just
# zlib package compilation instructions

build:
    cd zlib-* && ./configure $CONFIGURE_ARGS && make -j$(nproc)

package:
    cd zlib-* && make DESTDIR="$DESTDIR" install
```

### Custom Customizations & Flags

*   **Flag Ingestion:** Standard flags like `CC`, `CXX`, `CFLAGS`, `CXXFLAGS`, and `LDFLAGS` are predefined by the global master justfile to target musl. Use `{{env_var("VARIABLE")}}` within your recipe if you need to pass specific variables to configure scripts.
*   **Patch Application:** To apply patches listed under the `patches/` directory, write an extraction or patch recipe in your `build` target:
    ```just
    build:
        cd project-* && patch -p1 < ../patches/fix-bug.patch
        cd project-* && ./configure && make
    ```
