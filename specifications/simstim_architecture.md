# **Simstim: Declarative Developer Sandboxes**

Simstim is Freeside OS's native developer workstation subsystem. It maps lightweight, user-defined OCI runtime contexts directly onto our pristine, secure, and immutable musl-libc host.

Instead of silently polluting the host or forcing global runtime overrides, Simstim uses a localized, explicit design: developers opt into their runtimes via a .simstim configuration file, and jump into their workspaces using a fast CLI utility (simstim enter).

## **1\. The Architectural Challenge: The "Musl Wall"**

Freeside’s bare-metal host is engineered with a pure musl-libc toolchain to keep the base system stateless, highly stable, and free from corporate dynamic library bloat. However, modern development workflows present a major friction point:

* **Hardcoded Linkers:** Standard upstream precompiled language runtimes (Node.js, Python, Rust, Go) downloaded by package managers like mise or asdf are statically linked against glibc pathways (e.g., expecting /lib64/ld-linux-x86-64.so.2 to exist).  
* **The Crash:** Attempting to execute these binaries on a pure musl host results in an immediate crash.  
* **The Solution:** Simstim isolates these workflows within native, containerized environments. It bind-mounts your project code and home directory so you can compile and execute software exactly as you would on a standard glibc distribution, while keeping your bare-metal OS completely untouched.

## **2\. Directory-Based Declarative Configuration (.simstim)**

Each project workspace declares its runtime environment using a simple, human-readable .simstim file written in TOML. This file dictates which base OCI/Docker image to leverage for compilation and runtime, alongside any project-specific system packages, networking ports, and graphic bridges.

### **Example .simstim Configuration**

Developers configure their environment by uncommenting their target distribution or supplying their own custom-built container image:

\# .simstim \- Freeside Developer Sandbox Environment Config  
\# Initialize this template using 'simstim init'

\# Choose your development target runtime (only one image can be active)

\# \--- Arch Linux (Default, bleeding edge) \---  
image \= "docker.io/library/archlinux:latest"

\# \--- Fedora Workstation (Standard enterprise-friendly toolchains) \---  
\# image \= "registry.fedoraproject.org/fedora:40"

\# \--- Ubuntu LTS (Ideal for legacy LTS node deployments) \---  
\# image \= "docker.io/library/ubuntu:24.04"

\# \--- Custom/Corporate Internal Dev Image \---  
\# image \= "registry.internal.net/engineering/glibc-dev-base:v2"

\# \--- Custom Workspace Packages \---  
\# System-level packages required specifically for this project's sandbox.  
\# These will be fetched and cached inside the workspace-specific snapshot.  
packages \= \["neovim", "tmux", "postgresql-client", "clang", "cmake"\]

\# \--- Custom Network Ports & GUI \---  
\# ports \= \["8080:8080", "5432:5432"\] \# Fallback TCP port-forwarding to host interface (optional)  
\# gui \= true                         \# Exposes Wayland/X11 sockets for nested GUI apps (optional)

\[mounts\]  
\# Optionally expose extra host folders to the sandbox (outside $HOME)  
\# "/srv/data" \= "/srv/data"

\[env\]  
\# Override environment variables specifically within this sandbox shell  
CC \= "clang"  
CXX \= "clang++"

\[mise\]  
\# Control automatic mise behavior on enter  
auto\_activate \= true  
auto\_install \= true

## **3\. The Path-Traversal Resolution Algorithm**

When a developer triggers a command inside a directory, Simstim locates the configuration context by looking upward recursively. This allows developers to organize complex multi-repository projects underneath a single root-level .simstim configuration without needing to duplicate state files.

/home/case/src/  
├── .simstim                   \<-- Root config found and executed\!  
└── billing-api/  
    ├── src/  
    │   └── main.rs            \<-- Developer runs \`simstim enter\` here  
    └── Cargo.toml

### **Traversal Flow:**

1. Simstim begins its search for .simstim in the current working directory ($PWD).  
2. If .simstim is found, the lookup immediately succeeds, and parsing begins.  
3. If not found, Simstim climbs to the parent directory (../).  
4. **Upward-Only Restriction:** The search recursively continues up to / (system root). Under no circumstances does Simstim scan downwards into subdirectory trees.  
5. **Lookup Failure:** If the root directory is reached without finding a .simstim file, the command terminates with a clear, descriptive error and scaffolds an action plan:  
   error: No .simstim configuration file found in this directory or any parent directories.

   To initialize a new developer workspace in this path, run:  
       simstim init

## **4\. Operator Commands & UX (The CLI)**

The simstim client workspace program is a fast, ergonomic tool written in Rust that manages workspace boundaries, setups, and interactive entries.

                                \[User executes command\]  
                                           |  
                                           v  
                              /-------------------------\\  
                             \< Is .simstim config found? \>  
                              \\-------------------------/  
                                           |  
                    \+----------------------+----------------------+  
                    | Yes                                         | No  
                    v                                             v  
       /-------------------------\\                        \[Print error & exit\]  
      \<  Is workspace activated?  \>  
       \\-------------------------/  
                    |  
          \+---------+---------+  
          | Yes               | No  
          v                   v  
   \[Connect directly to   \[Trigger background activation\]  
    shared guest socket\]              |  
                              \[Start systemd-nspawn\]  
                                      |  
                              \[Launch guest-agent\]  
                                      |  
                              \[Connect to guest socket\]

### **Command Matrix**

| Command | Arguments / Flags | Privilege | Operational Role |
| :---- | :---- | :---- | :---- |
| simstim init | None | Unprivileged | Scaffolds a pre-populated .simstim configuration file in the current working directory, commenting out popular OCI bases. |
| simstim activate | None | Unprivileged | Instructs simstimd to construct snapshots, boot the background container workspace, and launch the guest multiplexer agent. Returns immediately. |
| simstim enter | \[cmd\] | Unprivileged | Connects directly to the active guest agent. If the container is inactive, it implicitly triggers activate before launching the interactive guest shell. |
| simstim stop | None | Unprivileged | Connects to the guest agent and issues a graceful teardown command, exiting all active sessions and container processes. |
| simstim kill | None | Unprivileged | Bypasses the guest agent completely. Instructs host-level simstimd to forcefully terminate the container process tree immediately. |
| simstim snapshot | create \<tag\>, list, restore \<tag\>, delete \<tag\> | Unprivileged | Manages manual, named workspace checkpoints utilizing raw Btrfs subvolume clones. |
| simstim gc | \--dry-run, \--force | Unprivileged | Scans and garbage collects stale, unreferenced Btrfs workspace snapshots and cache blocks. |

### **Interactive Shell Integration**

To make using Simstim feel natural, operators can query shell integrations to generate standard completions and shell shortcuts:

\# Generate fish shell completions  
simstim shell completion \--fish \> \~/.config/fish/completions/simstim.fish

\# Generate shell hooks to simplify workspace transitions  
simstim shell hooks \--fish | source

Using the hook, typing simstim without parameters inside a configured workspace will intelligently expand to simstim enter, dropping you into your glibc compiler shell instantly.

## **5\. Under the Hood: Lifecycle and Boot Optimization**

To achieve real-time response speeds (![][image1]) while supporting custom developer setups, Simstim organizes sandboxes using a three-tier Btrfs subvolume hierarchy.

\+--------------------------------------------------------------+  
|             LAYER 1: Immutable OCI Base Template             |  
|          /var/cache/simstim/bases/\[base\_image\_hash\]          |  
\+------------------------------+-------------------------------+  
                               |  
                               | (Btrfs CoW Snapshot)  
                               v  
\+--------------------------------------------------------------+  
|            LAYER 2: Durable Workspace Snapshot               |  
|            /var/cache/simstim/workspaces/\[ws\_hash\]           |  
|  \- Bootstrapped Shells, Git, & Local \`.simstim\` packages    |  
\+------------------------------+-------------------------------+  
                               |  
                               | (Btrfs CoW Snapshot)  
                               v  
\+--------------------------------------------------------------+  
|            LAYER 3: Transient Runtime Container              |  
|          /run/simstim/active/$USER/\[project\_hash\]            |  
|  \- Ephemeral runtime modifications, destroyed on session exit|  
\+--------------------------------------------------------------+

### **Phase 1: The Initial Image Import (Layer 1 Base)**

When simstim activate or enter is first invoked, Simstim pulls the requested upstream OCI/Docker layer (e.g., archlinux:latest) and unpacks it into a read-only base template: /var/cache/simstim/bases/\[base\_image\_hash\].

### **Phase 2: Workspace Hydration & Commit (Layer 2 Workspace)**

Before booting the interactive session, Simstim determines if a durable **Layer-2 Workspace Snapshot** exists for this configuration. If missing:

1. Simstim creates a temporary Btrfs subvolume from the Layer-1 Base.  
2. It initializes our guest hydration hooks: installs the operator's shell, basic extraction tools, git, and any custom system requirements declared under packages in .simstim.  
3. It triggers mise install inside this layer to pre-warm any local mise.toml runtime configurations.  
4. Once completed, Simstim freezes this subvolume as a **durable, read-only parent Layer-2 snapshot** at /var/cache/simstim/workspaces/\[ws\_hash\]. This survives system reboots and remains persistent.

### **Phase 3: Ephemeral Runtime Boot (Layer 3 Run)**

For every subsequent activation of the sandbox, Simstim performs an instantaneous Btrfs CoW snapshot of the durable Layer-2 workspace snapshot into /run/simstim/active/$USER/\[project\_hash\].

* Spawns systemd-nspawn with $HOME and /etc/resolv.conf bind-mounts.  
* The shell launches in under ![][image2].  
* On session exit or stop, the Layer-3 transient layer is completely discarded. The durable Layer-2 parent remains fully optimized and cached on disk.

## **6\. Shared System-Level Mise Cache & Automatic Activation**

To prevent developers from downloading duplicate glibc compilers, language runtimes, and dependencies across separate workspace container lifecycles, Simstim integrates a system-level caching pipeline coupled with automated on-entry environment hydration.

### **A. The Global Host-Guest Shared Cache Bind**

Without a unified cache, if a developer spins up three separate Simstim sandboxes targeting different directories (e.g., using different Arch Linux snapshots), running mise install node@22 inside each would download the Node.js runtime three distinct times.

To solve this, Simstim mounts a structured, shared volume onto the container at runtime:

HOST STORAGE                                              SIMSTIM CONTAINER  
/var/cache/simstim/shared-mise/  \======================\>  /home/$USER/.cache/mise/  
   ├── downloads/                                            ├── downloads/  
   ├── installs/                                             ├── installs/  
   └── shims/                                                └── shims/

1. **Host Allocation:** The host allocates a shared path at /var/cache/simstim/shared-mise/ structured to map directly to mise's standard layout.  
2. **Dynamic User Isolation:** To avoid ownership collision and file lock contention, user-specific subdirectories are generated underneath the shared cache folder, sharing downloaded source files while compiling distinct target paths.  
3. **Execution Flags:** During systemd-nspawn execution, the orchestrator appends:  
   \--bind-user=$USER:/var/cache/simstim/shared-mise:/home/$USER/.cache/mise

   This transparently substitutes the local sandbox container's .cache/mise with the high-speed, persistent, host-managed storage backend.

### **B. Declarative Directory Detection (Automated Hydration)**

Freeside operators expect instant feedback. When you jump into a simstim shell, you want your active toolchains resolved and injected before your prompt renders.

Simstim implements an automated scanning hook on startup:

\[simstim enter\]  
       |  
       v  
Check directory & parents for:  
  \- mise.toml  
  \- .mise.toml  
  \- mise.local.toml  
       |  
       \+------(Found)-------\> Bind-mount shared cache /var/cache/simstim/shared-mise  
       |                      Run: \`mise install \--yes\` (checks shared cache first)  
       |                      Export: Active paths injected into guest PATH  
       v  
Launch Guest Shell (fish/bash/zsh)

1. **Scan Phase:** Immediately after locating the project's .simstim file via the Upward Traversal Algorithm, the cli checks for the existence of mise manifest files (mise.toml, .mise.toml, or mise.local.toml) in the current directory tree.  
2. **Warm-up/Install Phase:** If auto\_install \= true is set in .simstim, Simstim invokes an ephemerally-contained mise install \--yes context prior to handing control to the interactive guest shell.  
   * If the required compiler version (e.g., Python 3.12.3) already resides in the shared system-level cache, it is loaded instantaneously (in ![][image3]).  
   * If missing, it downloads it securely once, storing it directly back into the persistent host-wide cache cache.  
3. **Shim/Path Injection:** Once hydrated, Simstim hooks mise env or auto-populates the container shell environment variables with the required /home/$USER/.cache/mise/shims paths, ensuring the target shell drops the developer directly into a fully active, fully functional runtime environment instantly.

## **7\. The Seamless Environment: Guest Bootstrap & Hydration Essentials**

Mounting directories is only half the battle. If a developer runs simstim enter and drops into an environment lacking their default shell, cryptographic credentials, and file extraction utilities, the system will feel broken.

Simstim runs a headless **Guest Hydration Engine** on the container's first boot to prepare a standardized developer blueprint.

### **A. Static Tool Injection (The Rust/Musl Superpower)**

We do not download or install mise inside the container using the guest’s package manager. Because the host version of mise is built as a statically linked musl binary on Freeside, **it has no dynamic link-time dependencies**.

We leverage this architectural trait by bind-mounting the host's static binaries directly into the sandbox namespace at runtime:

* **Mise Binary Passthrough:** We bind-mount /usr/bin/mise (host) to /usr/bin/mise (guest, read-only). It runs perfectly inside Arch, Fedora, or Ubuntu containers without compiling a separate glibc target.  
* **Straylight CLI Passthrough:** /usr/bin/straylight is similarly mounted, allowing developers to execute package operations or build recipes inside their workspace.

### **B. Host-Shell Mirroring & Resolution**

Developers expect their custom terminal environments (such as custom prompts, aliases, and completions) to follow them. Simstim reads the operator's active shell on the host via getpwuid (e.g., /usr/bin/fish).

To prevent shell initialization errors:

1. **The Check:** Simstim verifies if the host's default shell executable is present at the corresponding path inside the container rootfs.  
2. **Auto-Installation:** If missing, Simstim intercepts the boot flow and executes an automated headless package installation targeting the base OCI.  
   * **Arch Linux:** pacman \-Sy \--noconfirm fish  
   * **Fedora:** dnf install \-y fish  
   * **Ubuntu:** apt-get update && apt-get install \-y fish  
3. **Graceful Fallback:** If the network is unavailable or package databases fail to synchronize, Simstim prints a warnings and drops back to standard /bin/bash or /bin/sh to guarantee shell availability.

### **C. The Base Developer Utility Blueprint**

For modern web and software development, the environment requires a core group of foundational tools to support extraction, communication, and compiler toolchains. Simstim ensures the base container is populated with these essentials on first startup:

                  \+-----------------------------------+  
                  |      Simstim First Boot Hook      |  
                  \+-----------------+-----------------+  
                                    |  
                                    v  
                    Verify Guest Workspace Package Set:  
                    \- git (for repository management)  
                    \- openssh / gnupg (for secure transport)  
                    \- tar / xz / unzip / gzip (for extraction)  
                    \- make / patch (fallback runtime compilation)  
                    \- ca-certificates (trust stores)  
                                    |  
                   \+----------------+----------------+  
                   | Missing?                        | All Present  
                   v                                 v  
        Invoke Guest Package Manager            Inject user's host shell  
        to headless-install packages            & launch interactive tty

1. **Extraction Utilities:** tar, gzip, bzip2, unzip, xz, and zstd must be present. Because mise downloads zipped archives or tarballs of precompiled compilers, it depends directly on these guest utilities to unpack runtimes.  
2. **Git Integration:** Simstim verifies git is available. It maps the host's $HOME/.gitconfig and global git attributes so user identity carries over seamlessly.  
3. **Crypto & Credentials Forwarding:** \* **SSH Agent Forwarding:** The dynamic socket at $SSH\_AUTH\_SOCK is bind-mounted directly into the container. This allows git operations inside the container to authorize commits and fetches securely using host-managed SSH keys without ever exposing the raw private key files.  
   * **GPG Signing:** GnuPG sockets are passed through so git commit \-S operations can sign using the host's running gpg-agent.  
4. **Local Network Security:** The host’s dynamic /etc/resolv.conf and certificate directories are bind-mounted read-only to ensure SSL certificates (ca-certificates) trust corporate roots and secure registries out of the box.

## **8\. Cryptographic Addressing & Garbage Collection (GC)**

Spawning specialized workspaces containing dynamic packages and pre-compiled libraries runs the risk of storage decay. Over time, projects are deleted, configurations evolve, and older snapshots linger. To maintain a clean, compact host layout, Simstim enforces a strict **Cryptographic Addressing** and **Garbage Collection** protocol.

### **A. The Workspace Hashing Schema**

Durable Layer-2 Workspace subvolumes are located in /var/cache/simstim/workspaces/\[ws\_hash\]. The unique index \[ws\_hash\] is calculated deterministically based on three variables: the project path, the .simstim configuration, and the local mise manifests.

![][image4]Where:

* ![][image5] is the normalized absolute path of the project folder root on the host filesystem.  
* ![][image6] is the raw payload of the workspace's declarative .simstim file.  
* ![][image7] is the raw payload of any matching active mise configuration files (mise.toml, .mise.toml, or mise.local.toml).

If a developer alters .simstim (such as adding a system package to packages \= \[...\]) or updates a language runtime version in mise.toml, **a new, separate Layer-2 workspace snapshot is built**. The older snapshot immediately becomes unreferenced by that path.

### **B. Garbage Collection Rules**

Simstim indexes snapshots inside a lightweight local catalog database at /var/cache/simstim/catalog.db. This database keeps record of:

* workspace\_hash (Primary Key)  
* canonical\_project\_path  
* created\_at  
* last\_accessed\_at

The garbage collection routine simstim gc evaluates and prunes the workspace cache using three distinct criteria:

                          \[simstim gc triggered\]  
                                    |  
                                    v  
                       Iterate cached workspaces  
                                    |  
            \+-----------------------+-----------------------+  
            |                                               |  
            v                                               v  
 \[Is project path missing?\]                       \[Is configuration stale?\]  
            |                                               |  
            | Yes                                           | Yes  
            v                                               v  
 Prune Btrfs Subvolume                          If \> N historical hashes  
                                                exist for this path, prune  
                                                the oldest unaccessed layers

1. **Path Integrity Sweep:** Simstim checks if the registered canonical\_project\_path still exists on the host disk. If the developer has deleted or renamed the project directory, the corresponding cached Btrfs subvolume at /var/cache/simstim/workspaces/\[ws\_hash\] is destroyed instantly.  
2. **Stale Configuration Purging:** If multiple historical hashes exist representing the same project path (resulting from iterative configurations or changing dependencies), Simstim keeps the active hash and prunes the older, stale configurations, leaving a configurable generation margin (defaults to keeping the ![][image8] most recent builds for offline rollback).  
3. **Least Recently Used (LRU) Eviction:** A global storage cap can be defined on the host. When total cached workspaces exceed the cap, Simstim automatically evicts the oldest subvolumes based on last\_accessed\_at timestamps.

## **9\. Host-Guest UID Mapping and Mount Security**

One of the most complex pain points in containerized local development is file permission mismatches. Standard OCI runtimes run as root or map container users to elevated sub-UID blocks (e.g., mapping container-root to host-UID 524288 via User Namespaces).

If a container process writes a compilation artifact (like target/ or node\_modules/) under a mismatched UID, the host developer will experience immediate Permission Denied blocks when attempting to edit or delete those files in their local IDE.

To provide a flawless, integrated developer experience, Simstim enforces dynamic **Host-Guest Identity Translation** and **Namespace Coordination**.

### **A. Dynamic /etc/passwd and /etc/group Generation**

Rather than forcing standard static image configurations, when a user launches simstim enter, the client utility extracts the host environment's execution details:

* Real numerical User ID (![][image9]) (e.g., 1000\)  
* Real numerical Group ID (![][image10]) (e.g., 1000\)  
* Active username on the host (![][image11]) (e.g., case)  
* Default shell path on the host (![][image12]) (e.g., /usr/bin/fish)

HOST SYSTEM (musl)                                        SIMSTIM CONTAINER (glibc)  
UID: 1000, GID: 1000                                      UID: 1000, GID: 1000  
Username: "case"                                          Username: "case"

/etc/passwd  \========== (Dynamic Synthesis) \==========\>   /run/simstim/passwd  
"case:x:1000:1000..."                                     "case:x:1000:1000..."  
                                                                   |  
                                                                   v  
                                                          Bind-mounted over /etc/passwd

Before spawning systemd-nspawn, the parent simstimd daemon synthesizes a localized passwd and group configuration at /run/simstim/passwd and /run/simstim/group specifically mapped for the calling user. It appends the exact host user specification dynamically:

case:x:1000:1000::/home/case:/usr/bin/fish

To ensure files can be written and resolved without permissions collisions, these dynamically generated configuration files are bind-mounted directly into the container over /etc/passwd and /etc/group using the following nspawn flags:

\--bind-ro=/run/simstim/passwd:/etc/passwd \\  
\--bind-ro=/run/simstim/group:/etc/group

This guarantees that:

1. Standard tools inside the container (e.g., whoami, git, ssh) resolve the numerical host UID 1000 to the string name case.  
2. Home directory configuration matching is maintained automatically without configuration drift, and files written inside the container match the host user's permissions seamlessly.

### **B. Shared Host-Guest PID/UID Namespace**

To guarantee that compilation files are written back to $HOME with the exact permissions required by the host, **Simstim processes execute inside the container using the exact numerical user namespace of the host user**.

During sandbox setup, the execution engine passes the following arguments to systemd-nspawn:

\--user=case \\  
\--bind=$HOME:$HOME \\  
\--setenv=USER=case \\  
\--setenv=HOME=/home/case

* Because the container process is initiated as case (UID 1000), the Linux kernel validates that the container processes possess matching read/write credentials over the mounted host directories.  
* **No ID-Translation Layer (idmap) is Required:** By syncing the numerical IDs directly, raw performance is preserved, bypassing dynamic lookup tables at the filesystem layer.

## **10\. Persistent Session Multiplexing & PTY Bridge Protocol**

To ensure developer sessions survive local terminal crashes, connection drops, or emulator closures, Simstim implements an unprivileged, persistent dtach-style multiplexing socket bridge. This decouples interactive shell sessions from the lifecycle of the parent terminal.

Instead of passing host terminal file descriptors via SCM\_RIGHTS, the interactive shell session is isolated inside the container, and raw I/O bytes are tunneled securely over a local Unix domain socket.

\+--------------------------------------------------------------+  
|                     HOST MACHINE (unprivileged)              |  
|  \[Terminal Emulator\]                                         |  
|         | (Raw I/O)                                          |  
|         v                                                    |  
|  \[simstim enter\]  \<==== (Unix Domain Socket Stream) \=====+   |  
\+----------------------------------------------------------|---+  
                                                           |  
\+----------------------------------------------------------|---+  
|                     SIMSTIM CONTAINER (glibc)             |  |  
|                                                          |  |  
|    \~/.cache/simstim/sockets/\[ws\_hash\].sock \<=============+  |  
|                    |                                         |  
|         \[simstim-guest-agent\] (PTY Master)                   |  
|                    |                                         |  
|                    \+---- (Multiple Virtual PTYs)             |  
|                                |                             |  
|                    \[Guest Shells (fish/bash)\]                |  
\+--------------------------------------------------------------+

### **A. The Separation of Concerns**

* **The Lifecycle Daemon (simstimd):** Runs as a privileged service on the host, listening at /run/simstim/simstimd.sock. Its sole responsibility is orchestrating operating system namespaces (initiating, stopping, or destroying systemd-nspawn container workspaces and managing Layer-2/Layer-3 Btrfs snapshots).  
* **The Connection Agent (simstim-guest-agent):** Runs as an unprivileged developer-owned helper process *inside* the active guest container. It binds to a Unix domain socket exposed in the user's home directory: /home/$USER/.cache/simstim/sockets/\[ws\_hash\].sock.

Because the host user’s $HOME directory is bind-mounted directly into the container, **the socket file is visible to both the host client and the guest container processes**. This allows direct, high-speed, unprivileged communication between the host CLI and the guest workspace.

### **B. The Connection and Execution Lifecycle**

#### **1\. The Handshake Protocol**

When a developer invokes simstim enter inside a project folder, the client detects the canonical workspace hash (ws\_hash) and searches for an active guest socket at \~/.cache/simstim/sockets/\[ws\_hash\].sock.

1. **Activation Phase (If Socket is Missing):**  
   * The client connects to simstimd at /run/simstim/simstimd.sock.  
   * It sends the environment configuration and workspace targets.  
   * simstimd instantly clones the Layer-3 transient subvolume, configures the bind-mounts, and boots the systemd-nspawn container in the background.  
   * Systemd inside the container automatically triggers simstim-guest-agent inside the guest namespace. The agent binds to /home/$USER/.cache/simstim/sockets/\[ws\_hash\].sock and listens.  
2. **Terminal Handshake:**  
   * The client opens a direct stream connection to \~/.cache/simstim/sockets/\[ws\_hash\].sock.  
   * It puts the local host terminal into **raw mode** (disabling host-level handling of control characters like Ctrl+C or Tab so they pass transparently to the guest).  
   * It sends a serialized JSON configuration packet over the stream:  
     {  
       "pwd": "/home/case/src/billing-api/src/auth",  
       "env": {  
         "TERM": "xterm-256color",  
         "COLORTERM": "truecolor",  
         "GPG\_TTY": "/dev/pts/3"  
       },  
       "winsize": {  
         "ws\_row": 44,  
         "ws\_col": 132,  
         "ws\_xpixel": 0,  
         "ws\_ypixel": 0  
       }  
     }

#### **2\. Session Forking (Unrestricted Terminal Multiplexing)**

When simstim-guest-agent accepts the incoming stream connection:

1. **Virtual PTY Allocation:** It uses standard POSIX terminal interfaces (posix\_openpt, grantpt, unlockpt) to allocate a fresh, isolated pseudo-terminal (PTY master/slave pair) *inside* the container namespace.  
2. **Path Synchronization:** It forks a new child process. In the child context, it shifts directory execution by invoking chdir() directly onto the passed host "pwd" directory. Because home directory layouts are identical, this directory shift succeeds instantly.  
3. **Environment Injection:** It cleanses the child's environment block and replaces it with the passed host "env" mapping, ensuring credentials, terminal colors, and custom variables are mirrored.  
4. **Shell Execution:** It executes the user's mirrored host shell (e.g. /usr/bin/fish \--login) inside the slave terminal.

#### **3\. High-Speed Byte Copy Loop**

Once the child shell starts up, the guest-agent enters an asynchronous, zero-copy socket loop that forwards terminal streams:

* **Host Standard Input** ![][image13] **Unix Socket Connection** ![][image13] **Guest PTY Master** ![][image13] **Guest Shell Input**  
* **Guest Shell Output** ![][image13] **Guest PTY Master** ![][image13] **Unix Socket Connection** ![][image13] **Host Standard Output**

Because we are copying raw bytes over a high-speed Unix domain socket directly to a local guest PTY, latency is completely imperceptible (![][image14]).

#### **4\. Handling Terminal Resizing (SIGWINCH) without File Descriptors**

Because no host terminal file descriptors are passed across the boundary, standard host window resizes do not propagate directly to the guest kernel terminal emulator. Simstim handles this elegantly at the application layer:

1. When the operator resizes their host terminal emulator window, the simstim enter client captures the host-level SIGWINCH signal.  
2. It queries the new window size using ioctl(STDIN\_FILENO, TIOCGWINSZ, ...).  
3. It packages this resize state as a lightweight structured frame and sends it inline over the active Unix socket.  
4. The guest-agent parses the control packet, intercepts it, and issues an identical system call targeting its guest PTY master:  
   ioctl(pty\_master\_fd, TIOCSWINSZ, \&new\_winsize);

5. The containerized interactive shell (e.g., readline, fish editor) is notified immediately by the guest kernel namespace, updating prompt rendering and word wrapping in real-time.

## **11\. Interactive Session Persistence and Command Control**

Because the execution plane is anchored entirely inside the containerized guest-agent, the terminal sessions possess robust, persistent properties similar to standard system terminal multiplexers.

### **A. Persistent Workspace Multiplexing**

If a developer opens three distinct terminal windows on their host desktop and runs simstim enter inside the same workspace canonical path, each terminal command executes its own separate shell process inside the *same* running container workspace.

* They share the exact same filesystem, dynamic memory space, local ports, and compilation target caches.  
* If a developer drops their connection (e.g., their host terminal crashes or they log out of an SSH session), **compilation loops and running processes inside the sandbox do not terminate**.  
* Executing simstim enter again dynamically reconnects the developer to a new interactive terminal shell instantly.

### **B. Workspace Management Commands**

To manage this background execution layer, Simstim provides dedicated commands to check status, activate, or teardown workspaces cleanly.

                              \[CLI command triggered\]  
                                         |  
            \+----------------------------+----------------------------+  
            |                                                         |  
            v                                                         v  
    \[simstim activate\]                                         \[simstim stop\]  
            |                                                         |  
  Query simstimd socket                                       Connect to guest  
            |                                                 agent socket file  
  Boot nspawn container in                                            |  
  background with guest agent                                 Issue shutdown signal;  
            |                                                 agent exits gracefully  
  Host command returns                                                |  
  immediately                                                 Host simstimd tears down  
                                                              Layer-3 transient subvolume

#### **1\. simstim activate**

* This command initializes the environment without launching an interactive shell.  
* It contacts simstimd to construct snapshots and boot the systemd-nspawn background container.  
* The guest-agent boots up and begins listening at \~/.cache/simstim/sockets/\[ws\_hash\].sock.  
* The host command returns immediately, leaving the container ready to accept connections.

#### **2\. simstim stop**

* This command initiates a graceful shutdown of the developer workspace.  
* It connects directly to the guest-agent socket and sends a graceful termination packet.  
* The guest-agent signals all of its child shells with SIGHUP, waits for compilation background tasks to complete cleanly, releases dynamic socket bindings, and terminates.  
* The systemd-nspawn container exits cleanly. Upon termination, simstimd sweeps the transient Layer-3 subvolume from /run/simstim/active/.

#### **3\. simstim kill**

* Used as an administrative escape hatch if containerized processes block or hang the guest-agent.  
* The client bypasses the guest-agent socket completely and issues a force-kill directive directly to host-level simstimd at /run/simstim/simstimd.sock.  
* simstimd immediately invokes systemd's machine manager:  
  machinectl terminate \[deterministic\_uuid\]

  This immediately halts the sandbox container at the cgroups level and removes the transient Layer-3 mount.

## **12\. Passwordless Sudo Containment & Graphics Bridges**

To enable frictionless application installation and UI rendering inside the workspace without compromising the host's pristine security boundaries, Simstim utilizes user-namespace virtualization and dynamic socket projection.

### **A. Confined Passwordless Sudo**

During the Layer-2 workspace hydration phase, the privileged simstimd host daemon configures sudo privileges *exclusively inside the guest rootfs*:

1. **Targeted Provisioning:** simstimd injects a dynamic, user-mapped sudoer override directly to /etc/sudoers.d/simstim inside the container:  
   \# Generated dynamically by Simstim for unprivileged user escalation  
   case ALL=(ALL) NOPASSWD: ALL

2. **User Namespace Containment:** Because systemd-nspawn segregates PID, network, and mount layers, the guest container's root user (UID 0\) is mapped to an unprivileged kernel identity context on the host. Executing sudo pacman \-S tcpdump or modifying guest libraries in /etc is fully supported, yet the process is entirely barred from accessing physical host devices, Raw-Metal storage controllers, or system memory blocks.  
3. **Mount Write Symmetry:** File system transactions executing via sudo inside the home directories (/home/case) utilize mapped UID/GID matching (UID 1000 to 1000). This ensures modifications are saved under the correct unprivileged host owner, bypassing any file permission locks.

### **B. High-Performance Graphics Forwarding**

If developers are building graphical desktop software, Electron frameworks, game engines, or rendering toolkits (e.g. Wayland or X11 outputs), Simstim supports dynamic GUI capability mappings:

1. **TOML Configuration Activation:** When gui \= true is set in the project's .simstim manifest, simstimd appends high-performance graphical socket mounts at container boot.  
2. **Wayland Display Forwarding:**  
   * It identifies the active runtime environment socket (typically /run/user/$UID/wayland-0).  
   * It mounts this path directly to the guest:  
     \--bind=/run/user/1000/wayland-0:/run/user/1000/wayland-0

3. **X11 Fallback Forwarding:**  
   * It forwards the local raw X11 socket path at /tmp/.X11-unix/X0 and exposes the $DISPLAY environment variables, allowing legacy graphic packages to render using hardware acceleration.  
4. **Direct Render Access (DRI):** GPU access is mapped via device passthroughs (--bind=/dev/dri) allowing zero-copy OpenGL and Vulkan hardware rendering directly inside the guest window frame.

## **13\. Explicit Workspace Checkpoints (Snapshots)**

To complement Simstim's automatic, file-hashed Layer-2 drift detection, the CLI exposes explicit checkpoint controls. This allows developers to capture, restore, and preserve specific system environments on demand.

### **A. Manual Checkpoint Command Interface**

Manual snapshots exist parallel to the default Layer-2 pipeline, allowing developers to execute trial compilations, heavy library audits, or system upgrades safely:

\# Freeze the current Layer-3 transient running state as a named backup  
simstim snapshot create "pre-libc-upgrade"

\# List all manual snapshots cataloged for this canonical path  
simstim snapshot list

\# Force-revert the active container state to a specific tag  
simstim snapshot restore "pre-libc-upgrade"

\# Delete a manual checkpoint from storage  
simstim snapshot delete "pre-libc-upgrade"

### **B. Btrfs Snapshot and Cataloging Logic**

1. **Atomic Filesystem Freezing:** Taking a snapshot of an active running filesystem can result in broken file write boundaries or transaction database corruption. When snapshot create is triggered, Simstim initiates a coordinated split-phase freeze:  
   * **Phase 1 (Sync and Suspend):** The client sends a freeze signal over the guest-agent socket (\~/.cache/simstim/sockets/\[ws\_hash\].sock). The guest-agent issues a system-wide sync to flush pending memory pages, then uses the kernel fsfreeze API to pause active writes.  
   * **Phase 2 (CoW Snapshot Allocation):** With the filesystem paused, simstimd on the host instantly triggers a Btrfs Copy-on-Write snapshot of /run/simstim/active/$USER/\[project\_hash\] into a durable snapshot directory:  
     /var/cache/simstim/snapshots/\[ws\_hash\]/\[tag\]

     This clone allocates instantly (takes ![][image3]) and consumes 0 bytes of extra storage.  
   * **Phase 3 (Thaw):** The guest-agent releases the filesystem lock, resuming processing instantly.  
2. **Catalog Registration & Garbage Collection (GC) Exemption:**  
   * All manual snapshots are registered within the local tracking database /var/cache/simstim/catalog.db.  
   * These directories are flagged as **Durable Checkpoints**. They are explicitly exempted from Least Recently Used (LRU) automatic cache sweeps. They persist across reboots and config rebuilds until explicitly discarded via simstim snapshot delete.

## **14\. Nested Containerization & Podman Orchestration**

Developers must have the ability to run application containers (such as running a Postgres or Redis service locally via a podman-compose script) inside their Simstim workspace. Simstim achieves this through unprivileged nested containerization (Option A: Real Nesting), maintaining strict workspace boundary isolation.

\+-------------------------------------------------------------+  
|                     Freeside Bare-Metal Host                |  
|  \[straylight\]          \[systemd-networkd\]         \[btrfs\]   |  
\+----------------------------------+--------------------------+  
                                   |  
                                   v (systemd-nspawn)  
\+-------------------------------------------------------------+  
|                     Simstim Guest Container                 |  
|  \- Mounts /dev/fuse & allocates subuid range 100000:65536   |  
|  \- isolated network namespace (ve-\[ws\_hash\] \<-\> host0)      |  
|  \- \[simstim-guest-agent\]                                    |  
|         |                                                   |  
|         \+---\> \[Rootless Podman\] (Nested engine)             |  
|                     |                                       |  
|                     \+---\> \[Nested Container: Postgres\]      |  
|                     \+---\> \[Nested Container: Redis\]         |  
\+-------------------------------------------------------------+

### **A. Rootless OverlayFS and Device Passthrough**

Rootless Podman inside the guest container depends on the ability to compile, slice, and mount dynamic container layers without executing as raw system root.

1. **FUSE Overlay Support:** To enable high-performance rootless mounting inside systemd-nspawn, simstimd mounts the kernel's user-space filesystem driver:  
   \--bind=/dev/fuse

2. **Dynamic ID Allocation (subuid/subgid):** Podman partitions nested container processes inside custom user namespaces. During workspace boot, simstimd appends unprivileged subordinate ID ranges inside the container's virtual rootfs:  
   * /etc/subuid ![][image13] case:100000:65536  
   * /etc/subgid ![][image13] case:100000:65536  
     This allows the inner unprivileged Podman process to map up to ![][image15] subordinate users safely within the container namespace.

### **B. Cgroups v2 Delegation**

Systemd manages resource boundaries via Cgroups v2 on Freeside. To allow nested container engines inside Simstim to delegate resources (CPU, Memory limits) to their inner containers, simstimd boots the systemd-nspawn container with the delegation flag:

\--property=Delegate=yes

This passes sub-control namespaces directly to rootless Podman, allowing it to govern resources for its child layers unprivileged.

## **15\. Isolated Virtual Networking & NSS DNS Resolution**

To eliminate port-binding conflicts on localhost (such as launching multiple database instances on port 5432 across different workspaces), each Simstim workspace operates inside an isolated, private network namespace. It remains fully accessible from the host system utilizing zero-config domain names.

### **A. Virtual Ethernet Isolation (--network-veth)**

When a container is activated, the orchestrator allocates a custom network namespace:

1. **Kernel Pipe Pair:** It appends \--network-veth to nspawn. The Linux kernel establishes a virtual ethernet pair: a device interface named ve-\[ws\_hash\] on the host, and host0 inside the container.  
2. **Dynamic Private Addressing:** The host's system-level systemd-networkd possesses a default matching profile for ve-\* devices. It dynamically spins up a private bridge or local routing endpoint, serving Link-Local DHCP address blocks inside the private 192.168.123.x subnet. The guest container is assigned an isolated host-accessible private IP (e.g., 192.168.123.45).

### **B. Zero-Config Resolution via nss-mymachines**

Rather than polluting /etc/hosts with fragile and mutable IP mappings, Freeside integrates Name Service Switch (NSS) lookup bindings:

1. **Machine Registration:** When simstimd boots the systemd-nspawn instance, it assigns the container a deterministic hostname using the machine name specified inside .simstim (or defaults to the folder name, e.g. p50-box):  
   \--machine=p50-box

2. **NSS Lookup Handoff:**  
   * Freeside’s bare-metal host uses glibc’s Name Service Switch to resolve hostnames. Its default /etc/nsswitch.conf profile is pre-configured with the mymachines module:  
     hosts: mymachines resolve \[\!UNAVAIL=return\] files myhostname dns

   * When an operator opens their host browser or initiates an API call to http://p50-box:8080/, glibc intercepts the name request, hands name resolution to systemd's active machine registrar, resolves p50-box directly to 192.168.123.45 instantly, and bridges the connection natively with zero physical port clashes.

### **C. Local Port Forwarding Mappings**

If developers need to expose their container’s nested ports to the physical host interface (allowing other developers on their physical local area network to connect, or to easily bypass local browser DoH restrictions), the .simstim configuration exposes optional direct mappings:

1. **TOML Configuration Mapping:** Defining ports \= \["8080:8080"\] triggers direct NAT address translations.  
2. **Nspawn Port Bindings:** simstimd reads the array and configures systemd-nspawn:  
   \--port=tcp:8080:8080

   This exposes the private guest port directly on the host's physical loopback and LAN endpoints, allowing standard network traversals.

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEQAAAAZCAYAAACIA4ibAAADaElEQVR4Xu2XW4hNURzGz7iOhCE6GnPOPjNn6uSE4rhN0bi8uJVbISkSkRTRiHFpCkkSxTyYjEuJPEy8oFxyywzjMpFJZqIphEniRVEav397bf4ts2fO0cSk/dXXWutb/3X71tpr7x0KBQgQoLOhoKAgStLV1gWxWGyw4ziV8Ho0Gr0HV9ox/wWSyWQPFjmSBZeRfohEIqPtmNzc3IHUP6F+jZTj8XiEfKO0sUI7J9jpfkx2WchntzVY2Ct4EV6BLa0Zgr5fDNEa5dViYCqV6q71ToXCwsJBHOU9TPQ+XCG7b8f4gXab2jCkCQPOWtoUEz9R650CeXl5A4wRj+FajOlpx7QHP0PM3dFCekTrxKdEh9u0/k/BwvsyoZ1yIpjwIqQudky68DNEymbhh7XOJgwXnXbHSUfBRhN3Jj8/fwRpOayBVTEDqYM34FWKs3V/lKehn4a3HXdj5TGeqWN8wWR6EbwBPjK3fTc7JlP4GcJEi0V3LEOITxr9lJxQ0snwHXwmsd7dQr4aPiD+PEaFTZ8H0V6EzLzD4XBvys3yyJv6HMrX4PyfA7YGGYTgVWaQP3o0/OBnCFqR6LJISx8qOvM5qbSH8BNattIOmbjxSlusNfLz4EfPEAHzWWefot+Aw2Md961QogftCChDxmidMRPGkHKtEzdMdNod8DTmVItWrePMaWjRFzzaQtN2gpTpK075O2ym7jLpbk7dkF+9tAHzuJTCehqv7yhj/AyRbxBjSKXWJc7o2z2NfA3zuaXjPENC6rH2DNFvKMaf5bibLX0K38tj6dW3C4L702grrIMbxSg7JhN4hsgptOtYQAN1VVqjPN1MfIbSqjMxhLRYynJBM35K8olEog/5BdS/hse8NmlD7hF53mj8lAG2yAVlx6QDzxD6GGfXoe+Cz8lmKa0EvtUfZrStzdCQSaa8lHKFVy9w3M2+oLWMQKc5cC+dvJTFidN2TFugzQ6ZJOlUu45+s6m7QzpHynIayTcQu0SFZTnuG6ZOabKwo9KvfD0rbblojjldxpDPsEjFnIOlXvmPYf47ymBtOm8hFnWCgZvMBIXf0G465r/Fg7xa6XOzia+Ac706c9mLGV4f9Y77drqrtDdioOMa9NVoX2QTjSH7TJ38QlxyXDPa/fUIECBAgAABAgT42/gBQmkftIqrwYkAAAAASUVORK5CYII=>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAC4AAAAZCAYAAABOxhwiAAACz0lEQVR4Xu2VO2hUQRSGN/EJKj7j6r7u3YcGt7AwlYUgKawsLLQQjCIqYgq1EcTCzkIsfEW0sBAfKIKJkWDUqFiYRYPxERRMYaVFfGChJAFF4ndyZ+Jx3LuEFBuQ+8PP3PPPmXP/mdl7NhaLEGHCqPE8L+uKFplMpon5TliC17PZrOfmVBXpdDrh+/4GzHTBNndegN4MX+bz+cUSs4nDxAM2rjowsBfTfZhogT/KGedk4+jD5G5Sci3ae3hAaZMDTAyWM462A46wgZWOfp/N3NHapKCCcbmNEX4WaUdvhd+5sZlarzoqGG8T4xhconVO+5rofCN5xp3wk8Sw2XwzZ+AbeLpYLE5nbERvZ3wGW1lfVOVqmNtj5ru9oAGUcrncXJVTHl6IcYrdE0OucbQrxugKTOTgFhP3krtbchKJxCLiIebuwnMNDQ3TYkH36oUXbS3y18IeG2N4OfMDst5qoQgzjtYRYvyy6KlUapnEyWRyoTHe6eS9gl8LhcIMq7GJs/I+FR8nfsjjVKsR3+Cd82wcCilEgZtl9EvG0FJHvyq6GJZYXiIx41En7zl85GijXczGrNlm3tEPO4j3641WhBdinCInTNG//pwkV3QeayWur6+fY/KO6DwvMC6nqTUx/lNJ0l4PwSFTQw6gxx/Ph0/yIIntrq5OY5XWiR/DbhvX1dXNNnnjNS6bHgWHsN7enHQvOXHmfzFu/7MqBMb4LVen6HzmvjFutZrpEl/Q9lnN/lQmYpzn89Rq0jnyQVPzoNb+gfyeWDwMHxDWuPN+0N5KPE6RmKK7iF/oq6QTZMQM2smxhUEH6Sf/qdJk/QXJjcfjsyQW46zrs11Eug/aR1rtGr1uDEw2wifG9IjhO9glV69z+edch37MCz7WU3STBaqO9PHPqob04dXwrdJeG03eZ7UPGN7M2CK3J9+NH7Tf2zxv1O+PECFChAj/B34DEtHy/72f0dsAAAAASUVORK5CYII=>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADkAAAAZCAYAAACLtIazAAADAUlEQVR4Xu2WS2hTQRSGk/p+P0MgTXLbJBiJy4DgQnxs3LhQdKGgVKFSLIIvfCMoIlK6EGlFF4qioitFQRQVRBeK1lpRWrAbC+pCKwguLFgp8TvNjD0dQ5JWoRXuDz8z8885554zryQQ8OHDRyqVmhCPx896ntcNP8ArlZWVUdfuvwZFXYNN1dXVYdrV8Bvsovjpru2oQiKRmFFVVbWJ7hh3ToNilrGLj+kGlVYPc7BRmY4esPohkj5Bgi9gbSaTGe/aaGC7D7teFmSn1eSomiK7tO2IIxqNzjbFvYbb5J65NoWA7S4pCN8LVqM/yxTZrW1HDHJvSOaY7By7sQ6pwrUphmw2Ow6/Gn3/GC8xRT6UMW2tFGy0euZX0Z6GHbBJTgvtcvRbtK3wBguVGfhKIMjcVjP/BD4VypVSNn+CnZvk5XehjYBbkMa6NsOFeWlzJLXCjBNwgynyJXqd6JFIZC7jHubuiY8sGHJQbOAlGw/7pbDFjiluHvOfxN9qg2BWvs6sRtnHslwQez5x++BhrXNP55gi72rdy1+PrzoPCj6D9l2NT3r5U/F7Ixhf51sz7XgQeOYXevnfsj0YTXTn/was7GRJWt9PC0lIiqRt0DpaG3zkaM2w147xqRFf2AlvM95RcnPMUT0IO8ThHxUrx+wqPB8o8LOTTqenmUSPa90U2X93lSZF/lRShcm3x8SQxWopK2/zCh6Cr+BuKd61KRd88AjxbgbUwyXHzPZDodDUIRaZs2PirJTjLv1kMhmTjWG+j3bzgFcJmL9l23Fsx/FAOBye4toUA35rZGW1n7yY8fyfhH7Y4zqcIumfI9ZGbSOPFTH3a60sSCKwgaDvCLJXjphr4wK7BV7+b5y8iA9gK9p72h/wsrXjRYxL4sQ/pdzliHdi/1xpEvOi2NpFkyLxe2NfU3k80T7HYrHF2m9IkGBy/GR3Sl1wPtYoCRUi/keNjfxOflFz8rIvgm+V1m60Z0r7SIz1tM1y0uQ6ML7P+A79tW4uPnz48OHDh4/i+AWLse8ZhE4rpQAAAABJRU5ErkJggg==>

[image4]: <data:image/png;base64,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>

[image5]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAGcAAAAZCAYAAAAsaTBIAAAE50lEQVR4Xu2ZfWhXVRjHl71ZVJA0qr3cc7dmq1FEDWLaC7O0wLQklaIyTMjKtOyPao40s0mzovoj648w0tVEXdSIXkwkM1gFEQQx6Y0gi6D6w/6woRHr8/2d58Txtv12XVFr3S98Oc95nuec+zvne+85595fRUWBAgUKjC+kabrCOTdYhp/BteRNzLb9K6DPXeEa9N2SjReIwCQ9rYlKkuTujP8Gm8S+qqqq4+NYHjQ2Np5In/Oz/qampmMkeiFODjBB60ycm7Ix/M+ZQEuzsZFQV1d3Ke02Z/0C/rmFODkQxIE3ZmN6mmwSe7OxkUC7jjLiXFuIkwMjiHO/YpTbs7FyoM3ZcP9w4nDNawpxcqCcOPheMnGWBR/5t+F7D+7E3k35PPH6EMdebP0dwlgI6leZfwr+5ZSbafch9oqQU6BiaHEaGhpOwn8Lvl/gDlxHyd/c3Hw09QH4ScjFfgL2h5zI/70mPfYFROL0sTddKB/izJKPsimb/79FJE6glqM9TNJblO0IdWycr4MDnB3Vp9ukTo/zXA5x4icFkU7N+v7rYCynwdaamprjsrFciMRZko0NBYlF7kzbj56BXdZ+bpyXRxyVwcdx/RTzrYxz/01UVlaewO/5grHOyMbygHb30f5XxGnIxnLhcMUhv5fcg1x4AfZE7GlqT31enOdyiEP7K4NvLIoj8Hvura2tPSPrzwvNwz8ijibTJvX24KN+mfmuYxDnVNjeE4tDrAV2hjbkXT2cOPhWB994AGP69u8QZ8QXTec3/0H2h8bg0xMTxKHs0ZcB83+Db4tsxLiE+qqon/CeM5Q4DwVf6vE2/sfgJnh66p/WLso22MlvucL6nAo/5TovwEeIPUW9I/SF72Tq2yifpNygG8SuoYPPT/gfpHwYdmss1mYe9m64MPSj65LzhvrGvsfyZsFn8a2Ea+MvKtT3jlocZxOuH5eNZcGPmaNceKfq+hG02y5f6o/Yr+GeoJhs2G8nvI4k+pSjJdH6+WOfon1qvsdVt3Yf4261+EbqM3WSpOyRmNposX+UbTnL4efhEEPsK+pnmd2t68q2veRL2p9rsZfhJtlawrD3yrY+t1BfZLZuQJ1ew0Go2+xbna08JvKjob36Omxx0uE/fP7pfScG8UVwJ+xxXtgp8EX4NX3OifLOh985P0F6goJo6+EBu9aA7uTUC/uD+XSjvOv8k3AgtIuhwRJbAh+A+2w51cQsczZhAnY/fbfoacb+DdaFGLmvEmuzvK3Yd8iurq6uob4/6kNjK4lD+TpsD7EIR2rsNqevuGivdaMRZ6yDgbYysAHMI2K/lkjn7/rJqmvwLG3nmS3BukIudj+cmvgl7ZDlmPqu1JZQTSY5i2WbOLpuyNNpNIizQwKEWAC+XrhOtvohb2uIjUtxwASb3GmqaDlL/P62honYKJ8tffuoXy8SvysrDr6LzdbTXlpG7Z1KL9gXWEx7USzOwagPfSUpiUPOzc4va6UbhmsuTP0eOBhulsS/YmyLhB/9aW0sw9b/PueXz3btMVrCsD9w/sPqGrgBdjPhM1J/eJAg8ymXwp8lZH19fQLPpP6O83+RvE/ORbqG5eq/qzexL0/8gaF0MLHYHj0ZYYLxrXL+01VbYi/e2J3KgasT/7H4I8oFiX/P0RK+nvaT4rEVKFCgQIECBQqUxe9xfcYusxGq0wAAAABJRU5ErkJggg==>

[image6]: <data:image/png;base64,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>

[image7]: <data:image/png;base64,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>

[image8]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAaCAYAAACO5M0mAAABDUlEQVR4XmNgGHigoKBgD8Sn5OXlP0PpZHQ1DHJycsZAyVtA7CAtLS0D5K8AKvwPxCUoCoECu4GK3GF8FRUVdqDYfSD+BmSLwsSZQNYB8W0lJSV+mCDQ1BlQU+FOYAFyXoMEZWRkVGCCQH4fSAyoIRcmxqCoqKgPFHSCCzCAPbcLqtAZWRwFABUoAvFPID4C5DKiy8MBUMEioEmPQCGALgcHQCsjgQrfycrK6qDLwQEoPIGK7gMVmaLLoQCiFCorK8sCFd0BKrKFiQH5jkCnBMAVATkcoPgFuQ8uyABWWA20xRcuAOQsAAq+B+LdQHwAiK8D8RtQOAI1a4AVAeOSDySAC2tpabHBTRwFFAMA4BhIYH40B7MAAAAASUVORK5CYII=>

[image9]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEIAAAAZCAYAAACFHfjcAAADcklEQVR4Xu2XW0gVQRjHj1gPRVfIoqOe3aOnjnTDEoIKqofopTtSKlJml1dfhEhIqOgQShcfIoKUgrKgIiqKLhL1lARFBUFQIYFkRRC9GBhE/b6z38o6rJ7jhbZg//AxM///N7sz/52dnY1EQoQI4QfLslLEb58oFz0ajc7w0S4T1T68xAXbtht8eG/81LwSczyBg4G1yiBjsVidqRUVFU1FayS6iWVQua5G/l6d3DVPlzTgTqm2y+UKCgomYMBmuDfEl8LCwgXePoGDwbXooHebmoAJJMh5YPJwFdqv1UdrEg2ztpsa5sbQPhJdeXl5k0w9MHiM6H96XogRTOi+yWdjBFFtagKut0P1lKkFBtcIyp2mJlAj7pk83LZMRlBWmZoAfpr2/WpqgcFjRI2pCcQI9LsmjxFbszCi0tRcoHdJDteZb2qBIAsj5vgZAVc+SiOea06FqQUCBnNiKCPgS9DvmHw2RliD7BECtNeZcv4qGMhxNaLW1AQs3XloN334jK/GEJPMQeuTnHg8vpR8m/ozirdm4kjAtWbLuE1+SNDhqA7a9/MJv5hoN/lsNsvBjJCBqv6dZo5w9CkdKyNs54uWPhhmDTqtkkFRHjQ1AXwV+mEffjSfzzrVm11urIzgGrK6Xg7bCIHlvK+dEX06HuTCd7B8kwYvN6wciRGJRGIK/GfiG4ZMd3k25YVw74h6IoV2Xk6jokke3FXKk5RtnEo3Ck8ZtZ3N/ojkU99E1IqhcO1EI78KE917ZITlLP8e4nREj9H62bwiFzPS0+BmNZYz2YumZukGLAcnLy+Gwj1F65a9wavJkVv6cPKcK23yztHeI3XKSzE9pcpplPZ7MQ5un63nH8rVtNdrfoc1khUh0M/kbZ1cL/FEJzJglcCtQXtB/NJciQ/EATvzT9cnIoUJs7zXFKgRPW6b+5yRiSaTycmWc6+4R7vBvfbDbSF6qd+KOf9K40S3RmNE0MCcRQy+222LETJZfS3k69L/etJ+jHZI/lvg11rOj+Erol71tBH5+fkF7gr7b8DESn2MaJA6/EP3Cctqov6DWILeEtOTKe11tm741K8T1bJ65ZVxr/nPo7i4eCYDP0v0yetAbLCck+cjJr5SnqrULef3vhN9hfSjfoxoJhqZcJNsnsovJ9qI1rKysvED7xYiRIgQIYaNPx9ETZYZ2iNJAAAAAElFTkSuQmCC>

[image10]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEMAAAAZCAYAAABq35PiAAADiElEQVR4Xu2XW0iUQRTHze436CbWujq7Klj2kkhU0sWUCroRQRD00EMXiiyCCIqoKCKTwnrwJaQXC6wkKiwEJeqh7KWiCwQViSRZD5ImlGRE/Q7fDH0c/HJdxDXYPxy+mfOfmXPmP5edTUlJIokkBoq8vLzJxpht2D2sF/uYlZV1Oi0tbVIkEimmXO7awr3Gfmuj3Wy+DdqvrIN2FYw31R9/2IAES7B2kmzkWxQKhWaIn4RLqT/C2uAuqG4j8bfYCRYoLhXfC+EyMzNznDMjI2M6Y+7F34U9lgXwd0o4SHY9if3CaqiO0LwVSlZdi5GC76XlIprD3yxcdnZ2luZovsqKWKe5hEFWioTasW4SnKl5B/imeMUgRlhzArhq4aPR6ErNJQQkc8CuULXm/IA/2JcYxh6Ff4nBzgtpTmB3pMS+rrmEgETu2Mkc1pwfrF4eZ32F9tP3eX9iYLM0JwiHw7mW/6S5hMB4R0QS2qK5WEC/Z/GKYY+o8NI/8IgOGZwYrPo6zcWCWMQIujPk+DgxgtoMKZjDbZtQmebwNblkfdai2vR7TIImCldkx+yiOorvUewLQ23VbeMBYy3iiTBB+wNBhzKb0DnNOZDcfmnD7rman58/xs+ZGC7QIDHostnGvul8lOsGUYwq916KCQSewiQ/0LE16EVovFdp3O+MIDFk4sITd63fNxhiMMYyxvo+IDEEdFpjvEdXvYijeXyHgsTA/yoeMbgvlgiHNfn9svsY6gz+k1gNttFx+OcZ79fvLHaJtnOtv5h6pe1zn1+pacY7ciJ0OfyuvxFiAB2XG+8ybWOAfb5AC4034ad9iYHvrQTF5mgO3xPh+Fk2yi/if8Ye5Obmpimu1ngvYREsh3KrlIkzzsaK2nYF2Hvx8613fvI+5p74EnvAO8PB90dNLs5u7Ad2l4ALSHosk1rq2uK/gnVIQGs9WLM8vU3/f9RE3D36/rHj1hJvt5RlR1H/Zv2rsXeq7VfjXcKV9qhfNr6dJLHiFmM4ACGuMbGdUrZi9Fj/Bpmwr53siJ/4SilHKG/nc4JvJztqvrRxYtBmcV/CD3swgTolRq+U5XKn3OGOlewA2SncD+PhbqWnp08UP4JU4C+xbTplDDk6hYWFo12M/wIkvYkJvMEaZMWx87K6suLCWwEeUr+INbr/PPhuWBGOYKdwpdrxdlCv4nvcFyaJJJJIIonBxB+talb3lRjngwAAAABJRU5ErkJggg==>

[image11]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEQAAAAZCAYAAACIA4ibAAADrElEQVR4Xu2XaYhOURjHx5Y9lDGZ5b3vLIw9ZiJLIQllCx9sWQbli6RRQk2WSGRLtizxBUXZsi+RD5QoIuv4Ymq++OKLKaTx+899Tu6ceU0zdm/vv/6d53n+55x7n+eee869aWkppJBCYxAEwUZYnYDTpGdmZnZOoJ3w50k6kOQhJRuLxZb6Wl5eXge0MlgBhxBq5vdJOsTj8Z329Bf6mpCdnV1An2t+PGkRKcgCXxNUEFbPVT+etHAFoZ3va4IV5IofT1pECjLP1wQVBP1yNEaBehHbDW/BJ/A03OB05upIn/3EHsHr2PsUk4Zfqhh8SWw62g7siu89kD+OBhSkm18QJZSTkzNJdnFxcQv8bZpHPm0rK8R6+QUFBS21wvDPmN4De4ld875t3FWyv13hL4Kb2V5fQSyBi853x3Fubu4YF8MfoKKYvRx+ysjIaOt0rQSNoYj55mvOarjLxswiNtb1/6tQIlaQEl8T9HqgnYuEmtL/uZKGN+EBEh3oRPy7lmwdMs849XEFoV32bdqaeJz4A5pX0fiPgrm66v79eL1gwCa74YTHbhA+/WNerGdQO/HPJDHDtHfwTbS/j8gKqXNNtP6/qiDxcGXWfGg2GAwaoZujXetrAvGZge0HQmFhYXv8qeZqtYyizx3acvwmtE/he9lujA9XEB7G3ATaLykIc2i1aS9rXEEES+JeWt0kmhG/zn5R6AJ2oRfRTujDiVVhNlFhLdkJ0T7EVsMis7XCEu5bbOJ90V4H4V60kXmOEmstDbsTsVN2Mh12GzttZjw8HDaoP/ZkWKLCEjsGy9j72tS+Uj0IwteiEu5Ns89zO25ParJoXytINVziYtil7lvFVtAz/Lc6oRSjYP2InXb98QdpDqZa7GIOJNdHGqdPd/nMcwR/kWza4/hzZKenp7fDL1cBia2I27FNO9I9jCA83hu/QgQ7Xi9Ysh/gXVvStVaNFeQS2hpdEPca7R6SzHB98vPzc9CvEv8Mb0jX05UW2L9TlMwx2I21glQ6n3H7lbAV+gvMjWhnGbuS2BT4Aft8LPwnay49+JmC/Cuw1VThfBVESaugKl709cW/jbaO1RTTZ0AQ/og+hstNrylIVlZWtltx/x1IsH+CgqySHYTHfM0T14rEroJF6Dvp19v6jI/bARGEX9Cz0UbrVXJz/jfgVetCAgfhR70mcCL2Q3hLG7eesuwg/G24hz5M47C3wi2wjMQ3a5O1+FB4GB7SF3Xtq6WQQgoppPDb8BW7mTGEzFAhbQAAAABJRU5ErkJggg==>

[image12]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEcAAAAZCAYAAABjNDOYAAAD0UlEQVR4Xu2YSWhUQRCGJ+4buGAIZJmejDloDqIMuXlQo7iBoKIgghHB9aCCF0UFY1yiifsCigsmYIwiGjxEUXHFg6igiAuCeNAEPehBYiA5xK/yqsdOM0YNo5nAFPx011/V/er9r9/rngmF0pa2tP0rM8aMAafBLXAHbMrMzBwSiUTqJF5YWNgPrk3R5I//W2OOzcxdSnvBcuFweDH+FqmD2Dg3v9uMYsaDrxQ0Xfzc3NyBFLoT7hNotHlww8G1JIjTizkqwWcR25LMvQz/hXDUMtHJ7z6jmMugJgF/zBVHjKIPJEGcdlOB4uKI5eXlzU41cRrA0QS8rKgO4vB09ydLHAQo98XBn5lq4rwCb+V18mMU+cj1ydubLHGYZ4cvDtebllLi6IexDbyhv4F2Eu0AP08MfreIE41Gh9LuYSVdp70N5vi5whG/S3uDcfdoZ3jx1BenoKCgPwXVqEAurrKaRri5Kk4LbW1+fn6WcPh7wHu6fWye7jzvyDGaM1nmhI/ZHNMTxLFGQSsp7AFodQXyckQcuckJlqO/wOWysrIG438EZT9Hto99Ts4Z6/cocaxlZ2cPosBZ4I0Uyg5SZGNWHDkDWQ5/nntDOrbtF7jvjPutOCY493zBL3HzumrMNdnnOjUuvNXnxJxC44VZcSLON8l44rA61qgQS21OIvsTccTwLyZLHOap9blOjYt/8jkxVky23uQiyxk9myQSx+hTIbZQ/fU2J5GZ/ytOhtQDnvqBTo0BTVx8ic+zAqZooaMdTs45CcUhViw+sWH4X8FD3F42T44KxM5aP/KH5xzmPa+520AVmOvkLyJ+ReNH6A9XfjUogy8FdfBR/JMm+BbKq9ph5/ylmUCc5/p0MoSTXQZ7zaS7vNwTUjzfpZGWk3HCyenWcoxbLhzYIb/LQsFPBjlxx995+ockRz7gzrj5Om6mkyc7aZX0ucYoE+yMUuNYqTsWi/XVsWuJVYsfCc5n7bsnXIW0Irjpwsq5rU91I/1noAk0iB9SsfSH50MtXCDxEnCcfrNy30GlnZfYOskDn+nX0s7SUG9vrg+R4FUUseJzwZVrfTX0V0k/Jycn1+ghVOL0T+mcIlwR/jed/7EJ7uVwWI8PXRIn1U2ElZUofRWnWfq0+0C1zSNnKn6r7Lb0CyPBgfYwaNSHHxfH/O2ularGjVz0xGlRXg6WT0K6urn5g/jn5JU3zg9pxt7Vfxpi8C91bIczWI80/QbJeauefnH454ZQKnET7Hj14JKII6d9Fecm2A42wa/Q6WTHOgoqmGeBc5m0pS1taUtbKtsPUats3zc2FfEAAAAASUVORK5CYII=>

[image13]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABUAAAAYCAYAAAAVibZIAAAAeklEQVR4XmNgGAWjYOCBgoJCALoYxUBOTm4DEAuii1MEgAa6AF1bgS5OMZCXl+8BYit0cUoBM9C1K4G40tjYmBVdkiygoqIiCnTpemVlZTF0OXIBE9DArUAsiS5BNgAaFgzE0ejiFAGQt6kWjjCgqKiohy42CkYBDQEANxsO2GeuZ5QAAAAASUVORK5CYII=>

[image14]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEkAAAAZCAYAAAB9/QMrAAADp0lEQVR4Xu2XWUhVQRzGb9pK+2KWV+/c670h+hCV1ZNEGAW9RBRFLy1QEoRQGBW2WFDQTtHykPTWitBCO9lCUT5UZNH6EL0ELSBZSgpJ1O/vmbFhuud6pReR88HHOfPNf7Zv5szMCYUCBAjQExCJRBYrpa7DOlgTi8WUG9MZKBdztR4DBrcKPovH46MljWFVpD+bdCoUFBQMjkaj04k/Dhvd/G6N/Pz8oXR+Ga+Zbp4NVkw2g2vFmAWWnIH2Aa6ztH9A/ROI+QJPw9fwmxvTLZFIJLIY8E46/BiuKCoq6uvG2CBmOfyNWeMd/Rb13LC1VCD+crc3KTc3d4Q25zksx6x+bkwyEHtETOLTynP087CZ1dLf1v3QrU3CjCF0brusHAa0CCnDjUkFyl0Qkyg7xtYx/KzoeXl5cVv3g49JmWhPYBv8RJ3DaWc3vET6LizNzs4eyPMw2k2edcRssCsgXaS8iZT4F8qbvB12jC9YOQMIroBPqagMqbcbkw505/4xCe2U6LDQ1v2gkpuUIZu6fLbkNcEa046YRforvMpETBaN9/nSJp9+gamAdC35c+S9uLi4D+n9lD1o8pNCAglaqbyjOu3Pyg/UcUU6lsSkk6IzGeNs3Q8+JrUDkw5IXTxLLG2h1tYbjbbCtpaTkzNKmzbLxJCeKEaZdFJQYKrSJ0+6+0UqUM8J6Qgc6+hnRA+HwyNt3Q/apO+uLpBBSV1ZWVmDLK191WDITKPJRInGs1JLcsq+gT/hbVjNqppi4lNCf2ob4SsqXPM/ZsnSlY4p5yJI5y+KHkpzj1OpTdqjB9/RT/XXpBlGMybJ2Ky4QuV9NaIL26Le3pseZCOk0CZYD9eKeW5MZ6DBpbrxSbZO+gF8aGupoDyTmlxdEPX2n66YtFnSclHlfZ7OllVVSsx9nu9I9zLl0oLsSzS2msIvqaRSTgw3xg/a6CaeS4wmdyu0BqnTaOagkH3DaDa0Sc2uLlD6c+uCSVt0Osr7W5MvYLuZhtYS6qpJBtQ5TM/ae9n8ZCbcmGSgzFzK1IX07ZyyZaTr7UFJfXoA5zoKWkCvhT+STRB6tZSVi67RonoFm5NLwAQkRCNvl44Rk6TNchPDe0WkC5dcX8ipQP3b4KN0Tz85QejAXuVt5Ifkcmrno82Gv+hgldH0BfYeeqMejFCO9Vr0Er0i7f3ko5gDj/HeqrUWuE95/48NVuwdbdI16trKszbqXVeOyq+U3bcAAQIECBAgQIAAneMPtmslKuR1FZYAAAAASUVORK5CYII=>

[image15]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAD0AAAAZCAYAAACCXybJAAAD20lEQVR4Xu2WWWhVVxSGE0esVqsSopnOzVCDQaQaRByQQhEcXhxQaRso+qDggxbxwTxYLahgoVCwUVDRovalIohGnDUiGlFoiga1xQkHtLZQUXyoIPb7c9fOXR6j90SwD/H+sDh7/ftfa6+zzzr7nLy8HHJ4P1BaWjomiqLd2AnsoHw/X1ZWthB+dPALCgr6pVKp9dhYr8uGpHnw67E/sCPof8Gm+fkA6ixirgE7Iy356+KaDoFwDkEPwsKMl2F3GXYPGvzj2IuY7aipqenVnigBkuTBX4z9WFtb21M+da2WTnVmMrXVXYPdZn6R/PLy8knonrIRI7zuFSAaij0mcIXjmrVIZWVlqePUAS3YPyx0GFsS5jqDJHmo5SLzT5irMH+s6sFOep3VecD530mX9WmTcL2EapPA4U8lcJXXwR1Dm/Lc2yBJHjRN2N9BRy0T7KbPOs10cUg+D1xFRcUwuAauAwLXIRD9jv0V5+NAczRbsUmQJI9anVYtDL66QTfoHwT+VrvpTwKXCAR8ZDvYin1B0lNcm+Hrw/sUYMXOR7OX8W/Yr9g4r0mCzuZRizN/i+tPuD0CT47zqh1+stV9DtvoN6tD0NKVCsQeYA1QPdQajC+QaLvXwjVia5y/QcWUlJQM8rpsSJpHpzo1bGLuHnbZny8C3HWrvdHaOZ/xZjbjYmFhYV+vfQk65SzweXFx8eDAs9hS8b512MGRYSwwP9xi13o+G94mjx4A84+wUYFjfN9qnOG4UeL0OgTuFdg3Tgve8Dz+lxa8zvMe2k2LvRSf6wyS5LHuk+ZKVVVVb3FR+pV8QYdUBR0PrsR0pzPRMejAMNE5z7N784xXy2uBWdifbMJcJ1M7SfPQcW9E0jzqQLW30yj2mnTUNsX8JvlsQv+gYW6I5XrtBrYhSh8kLZ6jqDoFhydNsm/Nbz89U5lDsDkT+WYkycN1tvkvPQj8m+JDO3P9QT43XRA0eu8t9vVPWiC4HtFjf1rDfW3FfSaf8Uxsl9cwN9E03wQOdNeG+UI8kuRh7a/ihVdXV3+I/wz7N+Quy3y7xwcd49HitLmB6xC8E30QtiJcjZtvO38B2yDfZPksciiV+T/uFqVP4SuKN04FL7JC9gQuhqx51NZR+mReIF8bxHib5V3enimv7SbFNxYVFX1gufTtPp3150RgwY8R/4xdpajb2NK4Jkr/rm6J0v/O2pQd6AZ6DTczBf45dsfzHknyWJvuVy1cW8i7j/FkrxGsA/TreSNKf8s32Qb8/6CAg3GuS0M/Gdz0zjjfpcENf087fhrnuyzsEFoZ53PIIYcc3hX+AxaMYm9OGWm8AAAAAElFTkSuQmCC>