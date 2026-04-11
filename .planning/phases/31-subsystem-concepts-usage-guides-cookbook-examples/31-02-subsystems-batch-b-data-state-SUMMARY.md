---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 02
subsystem: docs
tags: [docs, diataxis, concepts, how-to, mappings, memory, storage, contracts, runs]
requires: [30-02]
provides: [concept-pages-batch-b, how-to-pages-batch-b]
affects: [docs/concepts/, docs/how-to/]
tech-stack:
  added: []
  patterns:
    - "Diataxis Concept + Usage Guide pairing per subsystem"
    - "Cross-link via relative markdown paths"
    - "Minimal examples use in-memory/SQLite defaults (no extras required)"
key-files:
  created:
    - docs/concepts/mappings.md
    - docs/concepts/memory.md
    - docs/concepts/storage.md
    - docs/concepts/contracts.md
    - docs/concepts/runs.md
    - docs/how-to/mappings.md
    - docs/how-to/memory.md
    - docs/how-to/storage.md
    - docs/how-to/contracts.md
    - docs/how-to/runs.md
  modified: []
key-decisions:
  - "Concept pages include supplementary H2 sections (e.g. 'Picking a backend', 'Migrations', 'Contracts vs. ordinary Pydantic models') where the 5-section template alone would fall below the 40-line minimum — the 5 required sections are always present"
  - "Memory Usage Guide uses RunEphemeralMemoryConnector for the minimal example so the snippet runs with a plain 'pip install zeroth-core', and documents the three optional extras (memory-pg, memory-chroma, memory-es) in 'Common patterns'"
  - "Storage Usage Guide example routes through RunRepository rather than raw SQL — matches how application code actually uses the storage layer"
requirements-completed: [DOCS-03, DOCS-04]
duration: 6 min
completed: 2026-04-11
---

# Phase 31 Plan 02: Subsystems Batch B (Data/State) Summary

Ten Diátaxis Concepts + How-to pages covering the mappings, memory, storage, contracts, and runs subsystems — the data/state slice of `zeroth.core.*`.

## Overview

- **Duration:** 6 min
- **Start:** 2026-04-11T20:40:36Z
- **End:** 2026-04-11T20:46:50Z
- **Tasks:** 2/2
- **Files created:** 10
- **Files modified:** 0

## What shipped

### Concept pages (5)

| File | Subsystem | Key types covered |
|------|-----------|-------------------|
| `docs/concepts/mappings.md` | mappings | `EdgeMapping`, `MappingExecutor`, `PassthroughMappingOperation`, `RenameMappingOperation`, `ConstantMappingOperation`, `DefaultMappingOperation`, `MappingValidator` |
| `docs/concepts/memory.md` | memory | `RunEphemeralMemoryConnector`, `KeyValueMemoryConnector`, `ThreadMemoryConnector`, `ConnectorManifest`, `ResolvedMemoryBinding`, `InMemoryConnectorRegistry`, `MemoryConnectorResolver`, `PgVectorMemoryConnector`, `ChromaMemoryConnector`, `ElasticsearchMemoryConnector` |
| `docs/concepts/storage.md` | storage | `AsyncDatabase`, `AsyncConnection`, `AsyncSQLiteDatabase`, `AsyncPostgresDatabase`, `SQLiteDatabase`, `Migration`, `create_database`, `GovernAIRedisRuntimeStores`, `RedisConfig` |
| `docs/concepts/contracts.md` | contracts | `ContractRegistry`, `ContractReference`, `ContractVersion`, `ToolContractBinding`, `StepContractBinding`, `ContractNotFoundError`, `ContractRegistryError` |
| `docs/concepts/runs.md` | runs | `Run`, `RunStatus`, `RunHistoryEntry`, `RunConditionResult`, `RunFailureState`, `Thread`, `ThreadStatus`, `ThreadMemoryBinding`, `RunRepository`, `ThreadRepository` |

### Usage Guide pages (5)

| File | Minimal example flow |
|------|---------------------|
| `docs/how-to/mappings.md` | Build an `EdgeMapping` with all four operation types and run it with `MappingExecutor` |
| `docs/how-to/memory.md` | In-process `RunEphemeralMemoryConnector` write + read with `MemoryScope.RUN`; doc of three optional extras |
| `docs/how-to/storage.md` | `AsyncSQLiteDatabase.connect() → RunRepository.initialize() → list_by_status() → close()` |
| `docs/how-to/contracts.md` | Register two Pydantic models with `ContractRegistry`, build a `StepContractBinding`, resolve via `ContractReference` |
| `docs/how-to/runs.md` | Create a `Run`, `transition(PENDING→RUNNING→COMPLETED)`, reload via `repo.get()` and walk history |

## Cross-linking

Every Usage Guide ends with a `## Reference cross-link` pointing at `../reference/python-api.md` (stub page generated in plan 30-02; Phase 32 will populate its anchors).

Concept cross-links:
- `mappings` → `graph`, `orchestrator`, `contracts`
- `contracts` → `mappings`, `agents`, `storage`
- `runs` → `orchestrator`, `storage`
- `memory` → `storage`, `agents`
- `storage` → `runs`, `memory`, `contracts`

## Memory extras documentation

`docs/how-to/memory.md` "Common patterns" section documents the three installable extras exactly as declared in `pyproject.toml`:

```bash
pip install 'zeroth-core[memory-pg]'      # psycopg + psycopg-pool + pgvector
pip install 'zeroth-core[memory-chroma]'  # chromadb-client
pip install 'zeroth-core[memory-es]'      # elasticsearch[async] 8.x
```

## Cross-connector / migration pitfalls discovered

1. **Alembic migration races** — every repository runs migrations in its own `initialize()` call. In multi-worker deployments two workers can race on a fresh database; the storage Usage Guide now calls this out explicitly.
2. **Connection lifecycle** — forgetting `await db.connect()` before calling a repository raises an opaque error; both how-to pages surface this as pitfall #1.
3. **SQLite vs Postgres drift** — repositories paper over most differences but hand-written SQL in downstream subsystems still has to use portable syntax. Noted as a pitfall.
4. **Memory scope confusion** — `MemoryScope.RUN` vs `MemoryScope.THREAD` mismatch silently returns `None`. Noted as memory pitfall #1.
5. **Missing `memory-pg` dependencies** — `PgVectorMemoryConnector` requires a Postgres server *and* the `pgvector` extension; installing the extra alone is not sufficient.

## Deviations from Plan

None — plan executed exactly as written.

## Verification

- All 10 files exist at the paths listed under `files_modified`
- Each Concept page contains `## What it is`, `## Why it exists`, `## Where it fits`, `## Key types`, `## See also`
- Each Usage Guide contains `## Overview`, `## Minimal example`, `## Common patterns`, `## Pitfalls`, `## Reference cross-link`
- `uv run mkdocs build` succeeds (non-strict)
- `grep "from zeroth\.\|import zeroth\." docs/concepts docs/how-to` — every import uses `zeroth.core.*`
- `docs/how-to/memory.md` contains all three of `memory-pg`, `memory-chroma`, `memory-es`
- Concept line counts: mappings=40, contracts=42, runs=40, memory=42, storage=41 (min_lines 40)
- Usage Guide line counts: mappings=59, contracts=73, runs=67, memory=69, storage=63 (min_lines 50)

## Mkdocs build warnings (not plan-02 scope)

`mkdocs build` produced warnings against `concepts/guardrails.md` and `concepts/identity.md` (links to `deployments.md`, `../how-to/guardrails.md`, `../how-to/identity.md`). These files are owned by plans 31-03 / 31-04 and are out of scope for this plan. Tracked as pre-existing — not my fix to make.

## Commits

| Task | Commit | Files |
|------|--------|-------|
| 1: mappings, contracts, runs | `d31ab93` | 6 files (3 concepts + 3 how-to) |
| 2: memory, storage | `b733531` | 4 files (2 concepts + 2 how-to) |

## Self-Check: PASSED

- docs/concepts/mappings.md: FOUND (40 lines)
- docs/concepts/memory.md: FOUND (42 lines)
- docs/concepts/storage.md: FOUND (41 lines)
- docs/concepts/contracts.md: FOUND (42 lines)
- docs/concepts/runs.md: FOUND (40 lines)
- docs/how-to/mappings.md: FOUND (59 lines)
- docs/how-to/memory.md: FOUND (69 lines)
- docs/how-to/storage.md: FOUND (63 lines)
- docs/how-to/contracts.md: FOUND (73 lines)
- docs/how-to/runs.md: FOUND (67 lines)
- commit d31ab93: FOUND in git log
- commit b733531: FOUND in git log

Ready for plan 31-03 (Subsystems Batch C — governance).
