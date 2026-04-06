# Codebase Structure

**Analysis Date:** 2026-04-05

## Directory Layout

```
zeroth/
├── src/zeroth/                  # Main Python package (all backend code)
│   ├── __init__.py              # Re-exports storage primitives
│   ├── agent_runtime/           # AI agent execution: config, prompt, providers, tools, threads
│   ├── approvals/               # Human-in-the-loop approval gates
│   ├── audit/                   # Append-only audit trail, evidence, verification
│   ├── conditions/              # Edge condition evaluation and branch resolution
│   ├── contracts/               # Versioned data schema registry
│   ├── deployments/             # Immutable deployment snapshots
│   ├── dispatch/                # Durable run worker with lease-based concurrency
│   ├── execution_units/         # Deterministic code/script execution, sandboxing
│   ├── graph/                   # Workflow graph models, versioning, persistence
│   ├── guardrails/              # Rate limiting, quotas, dead-letter, backpressure
│   ├── identity/                # Auth primitives: actors, principals, roles
│   ├── mappings/                # Data transformation between nodes
│   ├── memory/                  # Pluggable memory connectors for agents
│   ├── observability/           # Correlation IDs, metrics, queue gauge
│   ├── orchestrator/            # Core graph execution engine
│   ├── policy/                  # Capability-based runtime access control
│   ├── runs/                    # Run and thread state persistence
│   ├── secrets/                 # Secret resolution and redaction
│   ├── service/                 # FastAPI HTTP layer, auth, bootstrap
│   ├── storage/                 # SQLite + Redis backends, migrations
│   └── studio/                  # (Emerging) Workflow authoring control plane
│       ├── leases/              # Authoring lease management
│       └── workflows/           # Workflow CRUD for studio
├── tests/                       # pytest test suites mirroring src/ layout
│   ├── conftest.py              # Shared fixtures (sqlite_db)
│   ├── test_smoke.py            # Top-level smoke test
│   ├── agent_runtime/           # Agent runtime tests
│   ├── approvals/               # Approval service tests
│   ├── audit/                   # Audit repository tests
│   ├── conditions/              # Condition/branch tests
│   ├── contracts/               # Contract registry tests
│   ├── deployments/             # Deployment service tests
│   ├── dispatch/                # Worker, lease, recovery tests
│   ├── execution_units/         # Manifest, runner, sandbox, I/O tests
│   ├── graph/                   # Graph model, repo, validation tests
│   ├── guardrails/              # Rate limit, dead-letter tests
│   ├── live_scenarios/          # End-to-end scenario tests
│   ├── mappings/                # Mapping executor/validator tests
│   ├── memory/                  # Memory connector tests
│   ├── observability/           # Correlation, metrics tests
│   ├── orchestrator/            # Runtime orchestrator tests
│   ├── policy/                  # Policy guard, enforcement tests
│   ├── runs/                    # Run/thread model and repo tests
│   ├── secrets/                 # Secret provider/redaction tests
│   ├── service/                 # HTTP API tests (run, approval, audit, auth, RBAC, etc.)
│   └── storage/                 # SQLite, Redis, JSON tests
├── apps/
│   └── studio-mockups/          # Vue 3 + Vite frontend mockup
│       ├── src/
│       │   ├── App.vue
│       │   ├── main.ts
│       │   └── components/
│       └── dist/                # Built static assets
├── phases/                      # Implementation phase plans and artifacts
│   ├── phase-5-integration/
│   ├── phase-6-identity-governance/
│   ├── phase-7-transparent-governance/
│   ├── phase-8-runtime-security/
│   ├── phase-9-durable-control-plane/
│   └── phase-10-studio-shell-workflow-authoring/
├── docs/
│   └── specs/                   # Implementation specs (n8n references, etc.)
├── live_scenarios/              # Local scenario helpers
├── .planning/                   # GSD planning documents
│   ├── codebase/                # This directory (architecture analysis)
│   └── phases/                  # Phase planning docs
├── .agents/skills/              # Agent skill definitions (progress-logger)
├── pyproject.toml               # Package config, deps, tool settings
├── uv.lock                      # Locked dependencies
├── CLAUDE.md                    # Agent guidelines
├── AGENTS.md                    # Repo-specific agent instructions
├── PROGRESS.md                  # Implementation progress log
├── PLAN.md                      # Master spec (not implementation guide)
└── .python-version              # Python 3.12
```

## Directory Purposes

**`src/zeroth/graph/`:**
- Purpose: Workflow graph definition, lifecycle, and persistence
- Contains: Pydantic models, SQLite repository, serialization, versioning, validation, diffing
- Key files: `models.py` (Graph, Node types, Edge, Condition), `repository.py` (GraphRepository), `serialization.py`, `versioning.py`

**`src/zeroth/orchestrator/`:**
- Purpose: Core execution engine
- Contains: Single file `runtime.py` with `RuntimeOrchestrator`
- Key files: `runtime.py` (~775 lines, the largest single module)

**`src/zeroth/service/`:**
- Purpose: HTTP API surface, authentication, authorization, bootstrap wiring
- Contains: App factory, route modules, auth, RBAC, composition root
- Key files: `app.py`, `bootstrap.py`, `run_api.py`, `auth.py`, `authorization.py`

**`src/zeroth/agent_runtime/`:**
- Purpose: AI agent execution infrastructure
- Contains: Runner, provider adapters, tool system, prompt assembly, thread state, validation
- Key files: `runner.py`, `provider.py`, `tools.py`, `prompt.py`, `thread_store.py`

**`src/zeroth/execution_units/`:**
- Purpose: Deterministic code execution with sandboxing
- Contains: Manifests, runtime adapters, sandbox management, I/O, integrity verification
- Key files: `runner.py`, `sandbox.py`, `models.py`, `integrity.py`, `io.py`

**`src/zeroth/dispatch/`:**
- Purpose: Durable background execution with lease management
- Contains: Worker loop and lease manager
- Key files: `worker.py` (RunWorker), `lease.py` (LeaseManager)

**`src/zeroth/storage/`:**
- Purpose: Shared database backends
- Contains: SQLite wrapper, Redis config, JSON helpers
- Key files: `sqlite.py` (SQLiteDatabase, Migration, EncryptedField), `redis.py` (RedisConfig), `json.py`

**`src/zeroth/identity/`:**
- Purpose: Authentication/authorization primitives (leaf module)
- Contains: Single `models.py` with ActorIdentity, AuthenticatedPrincipal, ServiceRole, AuthMethod

**`src/zeroth/studio/`:**
- Purpose: Emerging workflow authoring layer (Phase 10, in progress)
- Contains: `leases/` (editing lease management), `workflows/` (workflow CRUD)
- Status: Early development, no `__init__.py` at package root

**`tests/service/`:**
- Purpose: Most extensive test directory -- HTTP API integration tests
- Contains: `helpers.py` (shared test bootstrap), per-feature test files
- Key files: `test_run_api.py`, `test_auth_api.py`, `test_rbac_api.py`, `test_tenant_isolation.py`, `test_durable_dispatch.py`, `test_e2e_phase4.py`, `test_e2e_phase5.py`

## Key File Locations

**Entry Points:**
- `src/zeroth/service/app.py`: FastAPI app factory (`create_app()`)
- `src/zeroth/service/bootstrap.py`: Dependency wiring (`bootstrap_service()`, `bootstrap_app()`)
- `src/zeroth/orchestrator/runtime.py`: Execution engine (`RuntimeOrchestrator`)
- `src/zeroth/dispatch/worker.py`: Background worker (`RunWorker`)

**Configuration:**
- `pyproject.toml`: Package deps, pytest/ruff config, Python 3.12 target
- `.python-version`: Python version pin (3.12)
- `src/zeroth/service/auth.py`: Auth config models (`ServiceAuthConfig.from_env()`)
- `src/zeroth/guardrails/config.py`: Guardrail settings (`GuardrailConfig`)
- `src/zeroth/storage/redis.py`: Redis config (`RedisConfig`)

**Core Domain Logic:**
- `src/zeroth/orchestrator/runtime.py`: Graph execution loop, node dispatch, policy enforcement, approval gates
- `src/zeroth/graph/models.py`: All graph data models
- `src/zeroth/runs/models.py`: Run and thread data models
- `src/zeroth/agent_runtime/runner.py`: Agent execution
- `src/zeroth/execution_units/runner.py`: Code execution
- `src/zeroth/conditions/branch.py`: Branch resolution
- `src/zeroth/policy/guard.py`: Policy evaluation
- `src/zeroth/dispatch/lease.py`: Lease management

**Testing:**
- `tests/conftest.py`: Shared `sqlite_db` fixture (in-memory via tmp_path)
- `tests/service/helpers.py`: Shared test bootstrap utilities
- `tests/runs/conftest.py`: Run-specific test fixtures

## Module Internal Structure Pattern

Each domain module follows a consistent internal layout:

```
src/zeroth/<module>/
├── __init__.py          # Public API barrel exports
├── models.py            # Pydantic data models
├── repository.py        # SQLite persistence (if applicable)
├── service.py           # Business logic (if applicable)
├── errors.py            # Module-specific exceptions (if applicable)
└── <specialized>.py     # Domain-specific files
```

## Naming Conventions

**Files:**
- `models.py`: Pydantic data models for the module
- `repository.py`: SQLite-backed persistence layer
- `service.py`: Business logic / coordination
- `errors.py`: Module-specific exception classes
- `__init__.py`: Public API barrel exports (re-export from submodules)

**Test files:**
- `test_<subject>.py`: e.g., `test_registry.py`, `test_guard.py`, `test_runner.py`
- Test directories mirror source: `tests/graph/` tests `src/zeroth/graph/`

**Directories:**
- Singular domain names: `graph`, `policy`, `identity` (not `graphs`, `policies`)
- Compound names with underscores: `agent_runtime`, `execution_units`, `live_scenarios`

## Module Dependency Map

Leaf modules (no internal Zeroth dependencies):
- `identity` -- auth primitives
- `memory` -- connector interfaces
- `mappings` -- data transformation

Mid-level modules:
- `storage` -- depends on `governai`, `cryptography`
- `graph` -- depends on `storage`, `mappings`, `governai`
- `contracts` -- depends on `storage`
- `conditions` -- depends on `graph`
- `secrets` -- depends on `execution_units`
- `policy` -- depends on `graph`, `runs`
- `runs` -- depends on `identity`, `storage`, `governai`
- `audit` -- depends on `identity`, `storage`
- `approvals` -- depends on `runs`, `audit`, `storage`
- `guardrails` -- depends on `runs`, `storage`
- `observability` -- depends on `runs`
- `deployments` -- depends on `graph`, `contracts`, `storage`
- `agent_runtime` -- depends on `contracts`, `memory`, `governai`
- `execution_units` -- depends on `secrets`, `contracts`

Top-level modules:
- `orchestrator` -- depends on `agent_runtime`, `execution_units`, `conditions`, `mappings`, `policy`, `approvals`, `audit`, `runs`, `secrets`, `graph`
- `dispatch` -- depends on `runs`, `storage`, `orchestrator`, `guardrails`
- `service` -- depends on everything (via bootstrap wiring)

## Where to Add New Code

**New domain module:**
- Create `src/zeroth/<module_name>/` with `__init__.py`, `models.py`, optionally `repository.py`, `service.py`, `errors.py`
- Add corresponding test directory `tests/<module_name>/` with `__init__.py`
- Export public API through `__init__.py`

**New API endpoint:**
- Add route registration function in `src/zeroth/service/<feature>_api.py`
- Register routes in `src/zeroth/service/app.py` via `register_<feature>_routes(app)`
- Add permission constants in `src/zeroth/service/authorization.py` if needed
- Wire any new dependencies through `src/zeroth/service/bootstrap.py`

**New graph node type:**
- Add node data model and node class in `src/zeroth/graph/models.py`
- Add to the `Node` discriminated union type
- Add dispatch handling in `src/zeroth/orchestrator/runtime.py` -> `_dispatch_node()`
- Add `to_governed_step_spec()` method

**New storage migration:**
- Add `Migration` object in the module's storage/schema file (e.g., `src/zeroth/graph/storage.py`)
- Migrations run automatically via `SQLiteDatabase.apply_migrations()` when repository is constructed

**New test:**
- Place in `tests/<module_name>/test_<subject>.py`
- Use `sqlite_db` fixture from `tests/conftest.py` for database-backed tests
- Use `tests/service/helpers.py` utilities for HTTP API tests
- All tests are async by default (`asyncio_mode = "auto"` in pyproject.toml)

**New Vue component (Studio mockup):**
- Add `.vue` file in `apps/studio-mockups/src/components/`

## Special Directories

**`.planning/`:**
- Purpose: GSD planning and codebase analysis documents
- Generated: By analysis tools
- Committed: Yes

**`phases/`:**
- Purpose: Implementation phase plans and verification artifacts
- Generated: Manually authored plans; artifacts generated during implementation
- Committed: Yes

**`.worktrees/`:**
- Purpose: Git worktrees for parallel development branches
- Generated: By `git worktree add`
- Committed: No (each worktree is a separate working copy)

**`live_scenarios/`:**
- Purpose: Runnable scenario helpers for local development/testing
- Contains: Python modules with scenario setup code
- Committed: Yes

**`apps/studio-mockups/dist/`:**
- Purpose: Built frontend assets
- Generated: By Vite build
- Committed: Yes (for deployment/preview)

**`apps/studio-mockups/node_modules/`:**
- Purpose: Frontend dependencies
- Generated: By npm/pnpm install
- Committed: No

---

*Structure analysis: 2026-04-05*
