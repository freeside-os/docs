# Wintermute Packaging Agent Guide

Wintermute is the autonomous package orchestration and maintenance agent for Freeside OS. It is designed to automate the import, creation, refinement, building, and validation of user-space packages under the `packages/` workspace repository.

This guide details the explicit workflows, workflows' triggers, architecture, and developer operations for Wintermute.

---

## 1. Architecture Overview

Wintermute is implemented using the Google ADK and runs a multi-agent cooperative architecture. The system consists of a master workflow coordinator ([agent.py](file:///home/dq/Code/freeside/wintermute/app/agent.py)) that delegates responsibilities to specialized agents:

1. **Triage Agent (`triage_agent`):** Parses user prompts, queries security feeds, and classifies requests into package-specific tasks.
2. **Recipe Refiner Agent (`refiner_agent`):** Modifies package manifest and justfile recipes to enforce Freeside standard structures (e.g. `$DESTDIR` prefix paths, strict permissions, UsrMerge).
3. **Build & Fixer Agent (`builder_agent`):** Manages the sandboxed container compilation loops, interprets build errors, applies source patches, and records upgrade notes.

---

## 2. Explicit Workflows

Wintermute supports six first-class explicit workflows:

### A. Create Package (`create`)
* **Trigger keyword:** `create`, `scaffold`, `scratch`, `new`
* **Implementation:** [create.py](file:///home/dq/Code/freeside/wintermute/app/workflows/create.py)
* **Description:** Used to build a new package recipe from scratch when there is no existing Arch Linux recipe or when starting clean.
* **Flow:**
  1. Activates a `scaffolder` agent to search for the package details online via Google Search.
  2. Resolves and downloads the latest release archive to calculate its SHA-256 hash.
  3. Scaffolds schema-compliant templates for `package.manifest` and `package.justfile`.
  4. Runs the `Recipe Refiner Agent` to sanitize instructions.
  5. Compiles the package using the `Build & Fixer Agent` inside the sandboxed container.

### B. Fix Package Build (`fix`)
* **Trigger keyword:** `fix`, `patch`, `debug`, `repair`
* **Implementation:** [fix.py](file:///home/dq/Code/freeside/wintermute/app/workflows/fix.py)
* **Description:** Targets an existing package whose compilation fails in the container sandbox. It loops automatically to diagnose, apply compiler flag workarounds/patches, and ensure compilation passes.
* **Flow:**
  1. Activates the `Build & Fixer Agent` to perform an initial build in the sandbox.
     * **Sandbox retention (`keep_sandbox`):** During debugging and iterative patches, the agent sets `keep_sandbox=True` to preserve build artifacts and compile incrementally. For the final verification run, it sets `keep_sandbox=False` (or omits it) to ensure a clean build from scratch.
  2. If compilation fails, it retrieves the build log and inspects errors against its troubleshooting lookup table:
     * **Redefinition of Inline Functions:** Injects `CFLAGS="-g -O2 -fgnu89-inline"` for legacy GCC code.
     * **Missing `argp`:** Adds `argp-standalone` to build dependencies and appends `LIBS="-largp"`/`LDFLAGS="-largp"`.
     * **Missing documentation tools:** Injects variables like `MAKEINFO=true` to bypass doc generators.
  3. Writes patches and registers them in `package.justfile`.
  4. Rebuilds and repeats until verification successfully passes.
  5. Records applied patch metadata in the package's `README.md` under `## Upgrade Notes`.

### C. Import Package (`import`)
* **Trigger keyword:** `import`
* **Implementation:** [import_pkg.py](file:///home/dq/Code/freeside/wintermute/app/workflows/import_pkg.py)
* **Description:** Imports an Arch Linux `PKGBUILD` and converts it automatically to a Freeside package recipe.
* **Flow:**
  1. Runs `import_pkgbuild` to query the Arch Linux GitLab repository and translate PKGBUILD properties.
  2. Falls back to the `scaffold_agent` if the target is not found.
  3. Runs refinement and sandbox verification.

### D. Upgrade Package (`upgrade`)
* **Trigger keyword:** `upgrade`
* **Implementation:** [upgrade.py](file:///home/dq/Code/freeside/wintermute/app/workflows/upgrade.py)
* **Description:** Upgrades package versions, adjusts source archive URLs, and rebuilds them.
* **Flow:**
  1. Bumps version and downloads the new source archive using `upgrade_package_version`.
  2. Refines build instructions and compiles.
  3. **Approval Flow:** Bypasses authorization for security patches but prompts human operators with `yes`/`no` or `approve`/`abort` confirmations for standard package upgrades before promoting.

### E. CI Review & Compliance (`review`)
* **Trigger keyword:** `review`, `check`, `validate`, `enforce`
* **Implementation:** [review.py](file:///home/dq/Code/freeside/wintermute/app/workflows/review.py)
* **Description:** A quality gate that runs validation on one or all packages in the workspace.
* **Flow:**
  1. Validates that all defined dependencies actually exist in the local workspace workspace (rejects package if missing dependencies are found).
  2. Triggers auto-fixes for minor issues (missing README, incorrect justfile targets).
  3. Compiles the package, patching if necessary, and produces a pass/fail report.

### F. Security Audit (`security_audit`)
* **Trigger keyword:** `security`, `cve`, `audit`
* **Implementation:** [security.py](file:///home/dq/Code/freeside/wintermute/app/workflows/security.py)
* **Description:** Scans installed packages against external CVE feeds and auto-upgrades/patches them.
* **Flow:**
  1. Pulls the vulnerability list using `query_security_feeds`.
  2. Identifies matching packages in the local workspace.
  3. Delegates upgrades to `UpgradeWorkflow` marked as high-priority security updates (bypassing approval).
  4. Posts structured notifications to configured webhook channels (e.g. Slack/Discord/Google Chat).

---

## 3. Running Wintermute Local Commands

Wintermute commands can be triggered locally using `just` targets, or inside the Docker container to avoid host contamination:

```bash
# Run a review on a package via just
just wintermute run "Review package zlib"

# Trigger an explicit build fix via just
just wintermute run "Fix package build curl"

# Create/scaffold a new package via just
just wintermute run "Create package libuv"
```

To run inside Docker (mounting the host workspace):
```bash
docker run -it --rm \
  --user "$(id -u):$(id -g)" \
  -v "/home/dq/Code/freeside:/home/dq/Code/freeside" \
  -w "/home/dq/Code/freeside/wintermute" \
  -e GEMINI_API_KEY="$GEMINI_API_KEY" \
  wintermute uv run adk run app "Review package zlib"
```
