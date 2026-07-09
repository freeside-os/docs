# Wintermute Packaging Agent Guide

Wintermute is the autonomous package orchestration and maintenance agent for Freeside OS. It is designed to automate the import, creation, refinement, building, and validation of user-space packages under the `packages/` workspace repository.

This guide details the explicit workflows, workflows' triggers, architecture, and developer operations for Wintermute.

---

## 1. Architecture Overview

Wintermute is implemented using the Google ADK and utilizes a clean architecture consisting of three layers:

1.  **CLI Layer ([cli.py](../../wintermute/app/cli.py)):** Built with Click. Handles terminal argument parsing, input presence checks, confirmation prompts, formatting colors (`click.style`), and interactive CLI execution loops.
2.  **Use Cases Layer ([use_cases.py](../../wintermute/app/use_cases.py)):** Encapsulates core business logic as async generators that yield progress tuples (`(type, message)`) upwards, making them fully decoupled from styling and terminal output constraints.
3.  **Agents and Workflows Layer ([workflows/](../../wintermute/app/agents/workflows/)):** Defines the cooperative Gemini-backed sub-agents and ADK workflows that execute compilation, auto-patching, and metadata refactoring.

### Cooperating Sub-Agents
*   **Recipe Refiner Agent (`refiner_agent`):** Modifies package manifest and justfile recipes to enforce Freeside standard structures (e.g. `$DESTDIR` prefix paths, strict permissions, UsrMerge).
*   **Build & Fixer Agent (`builder_agent`):** Manages the sandboxed container compilation loops, interprets build errors, applies source patches, and records upgrade notes.
*   **Scaffold Agent (`scaffold_agent`):** Searches online for package details and scaffolder templates when building a package from scratch.

---

## 2. Explicit Workflows: Graph vs. Dynamic

Wintermute organizes its agent execution pipelines into two types of ADK workflows depending on their complexity and branching requirements:

### A. Declarative Graph Workflows
Graph workflows inherit from `google.adk.workflow.Workflow` and declare their step transitions deterministically as edges in `model_post_init`. They are used for pipeline routines that execute sequentially:

#### 1. Create Package (`CreateWorkflow`)
*   **Implementation:** [create.py](../../wintermute/app/agents/workflows/create.py)
*   **Trigger Command:** `just wintermute cli create <pkg_name>`
*   **Flow:**
    1.  Activates a `scaffold_agent` to search online for source metadata and compute release archive checksums.
    2.  Scaffolds schema-compliant templates for `package.manifest` and `package.justfile`.
    3.  Runs the `Recipe Refiner Agent` to sanitize instructions and directories.
    4.  Runs the sandbox compilation loop via the `Build & Fixer Agent`.

#### 2. Import Package (`ImportWorkflow`)
*   **Implementation:** [import_pkg.py](../../wintermute/app/agents/workflows/import_pkg.py)
*   **Trigger Command:** `just wintermute cli import <pkg_name>`
*   **Flow:**
    1.  Downloads raw `PKGBUILD` files from the Arch Linux packaging repository.
    2.  Translates PKGBUILD instructions to Freeside package structure.
    3.  Passes converted files to the refiner and compiles them inside the sandbox.

---

### B. Imperative Dynamic Workflows
Dynamic workflows inherit from `google.adk.agents.BaseAgent` and override `_run_async_impl` to yield events dynamically using imperative python logic. This is necessary for workflows with conditional loops, user retries, or complex state variations:

#### 1. Fix Package Build (`FixWorkflow`)
*   **Implementation:** [fix.py](../../wintermute/app/agents/workflows/fix.py)
*   **Trigger Command:** `just wintermute cli fix <pkg_name>`
*   **Flow:**
    1.  Performs an initial sandboxed compilation in the container.
    2.  If compilation fails, it retrieves the build log and inspects errors against its failure signature lookup table:
        *   *Redefinition of Inline Functions:* Injects `CFLAGS="-g -O2 -fgnu89-inline"` for legacy GCC code.
        *   *Missing `argp`:* Adds `argp-standalone` to build dependencies and appends `LIBS="-largp"`/`LDFLAGS="-largp"`.
        *   *Missing documentation tools:* Injects variables like `MAKEINFO=true` to bypass doc generators.
    3.  Applies the corresponding patch or environment injection.
    4.  If compilation continues to fail, it exits with `compilation_failed` to let the Click CLI handle the operator suggestion prompt loop.

#### 2. Upgrade Package (`UpgradeWorkflow`)
*   **Implementation:** [upgrade.py](../../wintermute/app/agents/workflows/upgrade.py)
*   **Trigger Command:** `just wintermute cli upgrade <pkg_name>`
*   **Flow:**
    1.  Looks up the latest upstream version using the `upstream` module (fast Arch packages API search with Gemini search-grounding fallback).
    2.  Bumps version, updates hashes, and rebuilds inside the sandbox.

#### 3. Compliance Review (`ReviewWorkflow`)
*   **Implementation:** [review.py](../../wintermute/app/agents/workflows/review.py)
*   **Trigger Command:** Used by checks to inspect compliance.
*   **Flow:**
    1.  Validates that all defined dependencies exist in the workspace.
    2.  Checks for required files (e.g., README.md, package.manifest, package.justfile) and triggers automated compliance fixes.

---

## 3. Running Wintermute Commands

You can run Wintermute commands using the local Click interface:

```bash
# Enter the interactive prompt menu:
just wintermute cli

# Run check on all packages:
just wintermute cli check --all

# Create a package skeleton and compile:
just wintermute cli create zlib --group extra --version 1.3.1

# Fix a package sandboxed compilation:
just wintermute cli fix zlib
```
