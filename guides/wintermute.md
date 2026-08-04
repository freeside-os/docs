# Wintermute Platform Guide

Wintermute is the autonomous agentic development platform and harness for Freeside OS. It automates development, testing, maintenance, and release workflows across all Freeside components—including `packages`, `straylight`, `installer`, `simstim`, `freesidectl`, and `fstools`.

---

## 1. Platform Architecture

Wintermute is built on the Google ADK and structured into clean, modular layers:

1. **CLI & Manifest Runner ([cli.py](wintermute/app/cli.py)):** Built with Click. Provides component subcommands (`wintermute cli <domain> <workflow>`) and the unified Kubernetes-style manifest runner (`wintermute run <manifest.yaml>`).
2. **Engine Runtime ([engine.py](wintermute/app/engine.py)):** Manages `ExecutionSpec` dispatch, domain provider resolution, context loading, and execution events.
3. **Dynamic Domain Registry ([registry_domains.py](wintermute/app/registry_domains.py)):** Autodiscovers domain plugins under `app/domains/` and dynamic setuptools entrypoints. Validates requested workflows at runtime.
4. **Namespaced Tool & Skill Library ([registry.py](wintermute/app/registry.py)):** Stores tools and skills indexed by namespace (e.g. `straylight:*`, `packages:*`, `installer:*`).
5. **Rust Diagnostic Parser ([rust_diagnostics.py](wintermute/app/parsers/rust_diagnostics.py)):** Parses structured `cargo check` and `cargo clippy` JSON output streams into LLM context.

---

## 2. Platform Component Domains

Wintermute dynamically discovers and supports component domains:

| Domain | Subsystem | Supported Workflows |
| :--- | :--- | :--- |
| **`packages`** | Packages repository | `create`, `import`, `fix`, `upgrade`, `review`, `security_audit` |
| **`straylight`** | Package manager CLI & daemon | `check`, `clippy_fix`, `audit_rpc`, `benchmark` |
| **`installer`** | Freeside TUI installer | `check`, `qemu_boot_test`, `dps_uuid_audit` |
| **`simstim`** | Developer sandboxes | `provision`, `snapshot_verify` |
| **`freesidectl`** | Unified system control CLI | `scaffold_subcommand`, `completion_sync` |
| **`fstools`** | Packaging & indexing scripts | `schema_validate`, `caidx_audit` |

---

## 3. Declarative Manifest Execution (`wintermute run`)

Wintermute supports executing Kubernetes-style YAML manifests containing `apiVersion`, `kind`, `metadata`, and `spec`.

### A. `kind: Workflow` (Procedural Execution Blueprint)
```yaml
apiVersion: "wintermute.freeside.os/v1"
kind: Workflow
metadata:
  name: "straylight-check"
  description: "Runs cargo check on straylight"
spec:
  domain: "straylight"
  steps:
    - name: "check-straylight"
      domain: "straylight"
      workflow: "check"
```

### B. `kind: EventHandler` (Long-Running Event Listener)
```yaml
apiVersion: "wintermute.freeside.os/v1"
kind: EventHandler
metadata:
  name: "github-listener"
spec:
  source:
    type: "webhook"
    module: "wintermute.sources.github_webhooks"
  handlers:
    - event: "issues.opened"
      workflowRef: ".wintermute/workflows/triage-issue.yaml"
```

---

## 4. Operational Commands

```bash
# Run a declarative YAML manifest:
just wintermute run .wintermute/workflows/straylight-check.yaml

# Enter interactive CLI menu:
just wintermute cli

# Run check on all packages:
just wintermute cli check --all

# Run fix on a package build:
just wintermute cli fix zlib
```
