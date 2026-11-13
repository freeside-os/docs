# Wintermute Platform Guide

Wintermute is the autonomous agentic development platform and harness for Freeside OS. Built on the **Google Antigravity Python SDK** (`google-antigravity`), it automates development, package maintenance, testing, troubleshooting, and documentation workflows across all Freeside repositories (`packages`, `straylight`, `installer`, `fstools`, `wintermute`, and `docs`).

---

## 1. Hexagonal Architecture

Wintermute follows a strict hexagonal architecture inside [wintermute/app/](../../wintermute/app):

1. **Domain ([wintermute/app/domain/](../../wintermute/app/domain)):** Pure core entities (`Task`, `WorkUnit`, `SubagentRun`, `PipelineTemplate`, `StageSpec`, `KnowledgeEntry`) and token/USD cost calculators (`cost.py`).
2. **Application Services & Ports ([wintermute/app/application/](../../wintermute/app/application)):**
   - **Pluggable Pipeline Engine ([pipeline.py](../../wintermute/app/application/pipeline.py)):** Executes declarative stage-gate pipelines with SDK `@hooks.stop` gates and pre-commit verification enforcement.
   - **Single-Agent & Swarm Router ([single_agent_service.py](../../wintermute/app/application/single_agent_service.py)):** Routes focused tasks via Fast-Path directly to execution, or escalates multi-domain/complex tasks through `tech_lead` `WorkUnit` decomposition.
   - **Cost Controller ([cost_service.py](../../wintermute/app/application/cost_service.py)):** Enforces task and daily USD budgets with pause/top-up resumption (`PAUSED_BUDGET`).
   - **Knowledge Service ([knowledge_service.py](../../wintermute/app/application/knowledge_service.py)):** Provides context priming, explicit/automatic note capture, and target consolidation.
3. **Outbound Adapters ([wintermute/app/adapters/outbound/](../../wintermute/app/adapters/outbound)):**
   - **Antigravity SDK Harness ([antigravity_harness.py](../../wintermute/app/adapters/outbound/antigravity_harness.py)):** Maps role definitions to `LocalAgentConfig`, `LocalOpenAIAgentConfig`, or `LiteRTAgentConfig` with exact allowlist policies, raw build-command guards, subagent wiring, `@hooks.pre_tool_call_decide` worktree materialization, `@hooks.on_interaction`, and session continuation (`conversation_id`).
   - **Just AST Synthesizer ([just_synthesizer.py](../../wintermute/app/adapters/outbound/just_synthesizer.py)):** Parses `just --dump --dump-format json` into typed Python `just_*` tools scoped per role and domain, serialized by per-recipe concurrency semaphores (`build_locks`).
   - **Worktree Manager ([worktree_manager.py](../../wintermute/app/adapters/outbound/worktree_manager.py)):** Manages ephemeral per-task workspaces (`.wintermute/worktrees/<task_id>` with lazy `git worktree add` or Btrfs CoW snapshots, plus `--no-worktree` direct mode) and isolated `build/workspace/<task_id>` outputs.
   - **SQLite Storage ([sqlite_storage.py](../../wintermute/app/adapters/outbound/sqlite_storage.py)):** Unified persistence for tasks, work units, subagent runs, audit events, cost ledger, model pricing cache, and hybrid `FTS5` + `sqlite-vec` memory.
   - **Auto-Pricing Catalog ([pricing_catalog.py](../../wintermute/app/adapters/outbound/pricing_catalog.py)):** Syncs live model rates from `models.dev` with OpenRouter fallback and hardcoded emergency rates.
4. **Inbound CLI & REPL ([wintermute/app/adapters/inbound/cli/main.py](../../wintermute/app/adapters/inbound/cli/main.py)):** Click + Rich terminal interface supporting interactive REPL conversations and non-interactive `exec` runs.

---

## 2. Configuration, Roles & AgentSkills

Wintermute uses a clean three-tier configuration model:

- **Platform & Pipeline Config ([wintermute.toml](../../wintermute/wintermute.toml) + `.wintermute/config.toml`):** Defines `[models.*]` profiles (Gemini, Vertex, local OpenAI-compatible, LiteRT), `[agents.*]` defaults and token/call budgets, `[cost]`, `[memory]`, `[concurrency.build_locks]`, and `[pipeline]` templates. Secrets and runtime environment overrides (such as `GEMINI_API_KEY` or `WINTERMUTE_MAX_TASK_USD`) are automatically loaded via `python-dotenv` from `.env`, `.wintermute/.env`, or `wintermute/.env` without overriding variables already exported in the shell.
- **Role Definitions ([wintermute/agents/](../../wintermute/agents)):** Markdown files (`<role>.md`) with YAML frontmatter controlling `description`, `read_only`, default `skills`, `allowed_subagents`, `allowed_tools` (supporting `@domain`, `@read_only`, and globs), and `denied_tools`, paired with the role's system prompt body. Project-local overrides live in `.wintermute/agents/<role>.md`.
- **Portable Skills ([wintermute/skills/](../../wintermute/skills)):** Follows the `agentskills.io` specification (`SKILL.md` with strict `name` and `description` frontmatter) passed via `skills_paths` for progressive disclosure and stage instruction injection.

---

## 3. Pluggable Stage-Gate Pipelines

| Template | Default Domains | Stages & Behavior |
| :--- | :--- | :--- |
| **`package_lifecycle`** | `packages` | `scaffold_or_edit` -> `static_verify` (`pkg::verify`, `pkg::resolve`) -> `sandbox_build` (`pkg::build`) -> `troubleshoot_build` (on fail only) -> `compliance_review` -> `reverify_after_review` -> `commit` |
| **`tdd_coding`** | `straylight`, `fstools`, `wintermute` | `implement` -> `verify_tests` (`@domain_test`) -> `troubleshoot` (on fail only) -> `review` -> `retest_after_review` -> `commit` |
| **`standard_change`** | `docs` | `edit` -> `review` -> `commit` |
| **`investigation_audit`** | Explicit (`investigation_audit`) | Read-only `investigate` stage (`read_only = true`, no edits/commits) |

A work unit **cannot** commit if files were modified after the last passing `GATE` stage, guaranteeing verified commits.

---

## 4. Tool Synthesis & Guardrails

Agents never run raw `cargo`, `pytest`, `straylight`, or `fstools` shell commands directly—`run_command` policies block those invocations and route agents through synthesized `just_*` tools (such as `just_pkg_verify`, `just_pkg_build`, `just_straylight_test`, `just_fstools_test`). Before executing writes or `just` targets affecting a sub-repo, Wintermute lazily materializes that sub-repo in the task worktree.

Package recipes also integrate directly with [fstools/src/fstools/pkg.py](../../fstools/src/fstools/pkg.py) and [just/pkg.just](../../just/pkg.just) (`verify`, `verify-all`, `resolve`, `build`, `checksum`, `audit`, `logs`, `create`).

---

## 5. CLI & Operational Usage

First, create a git-ignored `.env` file at the Freeside root from [.env-example](../../.env-example) and set `GEMINI_API_KEY` (or export it in your shell):

```bash
cp .env-example .env
```

Use the root `just` wrapper (`just wintermute ...`) or run `uv run --directory wintermute wintermute ...`:

```bash
# Start an interactive REPL session (or continue the most recent stored session)
just wintermute chat
just wintermute chat --continue
just wintermute chat --resume task_abc123

# List, inspect, resume, rename, or delete stored cross-process sessions
just wintermute sessions list
just wintermute sessions show latest
just wintermute sessions resume task_abc123
just wintermute sessions rename task_abc123 "zlib Musl Fix"
just wintermute sessions delete task_abc123

# Execute a single prompt non-interactively
just wintermute exec "Patch CVE-2025-1234 in zlib" --domain packages --target zlib

# Execute directly in the workspace without creating a git worktree
just wintermute exec "Update README notes" --domain docs --no-worktree

# Resume a paused or existing task (optionally granting extra budget) or retry a WorkUnit
just wintermute exec --resume task_abc123 --budget-top-up 5.0
just wintermute exec --work-unit wu_xyz789 --from-start

# Inspect configured pipelines, models/pricing, and role-scoped tools
just wintermute pipelines
just wintermute models
just wintermute tools --role senior_coder --domain packages

# Search, inspect, add, or consolidate hybrid knowledge entries
just wintermute memory search "musl static link"
just wintermute memory add "zlib musl build" "Set LDFLAGS=-static" --domain packages --target zlib
just wintermute memory consolidate --domain packages --target zlib

# Run linter and test suite
just wintermute lint
just wintermute test
```

Inside `wintermute chat`, slash commands provide live inspection and control: `/sessions [n]`, `/resume <id|prefix|#n|latest>`, `/rename <title>`, `/new`, `/status`, `/fast-path [on|off]`, `/wu-resume [id] [hint]`, `/wu-retry [id] [hint]`, `/history [n]`, `/retry [n]`, `/pipeline`, `/cost`, `/budget +<amount> | daily [+]<amount>`, `/diff`, `/tools [role] [domain]`, `/memory [query]`, `/help`, and `/exit`.
