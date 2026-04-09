---
phase: 14-memory-connectors-container-sandbox
plan: 01
subsystem: memory
tags: [memory, governai, protocol, connectors, settings]
dependency_graph:
  requires: []
  provides: [governai-memory-protocol, memory-config-settings, scoped-auditing-wrappers]
  affects: [agent-runtime, memory-registry, live-scenarios, e2e-tests]
tech_stack:
  added: []
  patterns: [governai-memory-connector-protocol, scoped-memory-wrapping, auditing-decorator]
key_files:
  created:
    - tests/memory/test_resolver.py
  modified:
    - src/zeroth/memory/connectors.py
    - src/zeroth/memory/models.py
    - src/zeroth/memory/registry.py
    - src/zeroth/memory/__init__.py
    - src/zeroth/config/settings.py
    - src/zeroth/agent_runtime/runner.py
    - tests/memory/test_connectors.py
    - live_scenarios/research_audit/bootstrap.py
    - tests/service/test_e2e_phase5.py
decisions:
  - Connectors follow GovernAI DictMemoryConnector storage layout (_store[scope.value][target][key])
  - Resolver wraps with AuditingMemoryConnector first, then ScopedMemoryConnector (audit sees raw scope, scoped resolves target)
  - AgentRunner uses MemoryScope.RUN for all memory operations (ScopedMemoryConnector handles target resolution)
metrics:
  duration: 583s
  completed: "2026-04-07T08:33:41Z"
  tasks: 2
  files: 9
---

# Phase 14 Plan 01: GovernAI Memory Protocol Rewrite Summary

Rewrote Zeroth's entire memory system to implement GovernAI v0.3.0 async MemoryConnector protocol with ScopedMemoryConnector + AuditingMemoryConnector wrapping, updated AgentRunner to use new async signatures, and added config sub-models for future memory backends.

## What Changed

### Task 1: Rewrite memory connectors, models, and registry

**Connectors** (`src/zeroth/memory/connectors.py`):
- Removed old `MemoryConnector` Protocol class and `_BaseDictConnector` base class
- Created three new async connector classes: `RunEphemeralMemoryConnector`, `KeyValueMemoryConnector`, `ThreadMemoryConnector`
- Each implements GovernAI's 4-method protocol: `read`, `write`, `delete`, `search`
- Storage layout mirrors GovernAI's `DictMemoryConnector`: `_store[scope.value][target][key] = MemoryEntry`
- All three pass `isinstance(connector, GovernAIMemoryConnector)` check

**Models** (`src/zeroth/memory/models.py`):
- Removed `MemoryContext` class entirely (replaced by GovernAI MemoryScope + target)
- Removed `ConnectorScope` enum (replaced by GovernAI `MemoryScope`)
- Updated `ConnectorManifest.scope` to use `MemoryScope`
- Updated `ResolvedMemoryBinding` to remove `context` field

**Registry** (`src/zeroth/memory/registry.py`):
- `MemoryConnectorResolver` gains `audit_emitter` and `workflow_name` parameters
- `resolve()` wraps raw connector with `AuditingMemoryConnector` (if emitter provided) then `ScopedMemoryConnector`
- Thread binding recording preserved

**Settings** (`src/zeroth/config/settings.py`):
- Added `MemorySettings`, `PgvectorSettings`, `ChromaSettings`, `ElasticsearchSettings` sub-models
- Wired into `ZerothSettings` with defaults

### Task 2: Update AgentRunner memory integration

- `_load_memory()`: `await binding.connector.read("latest", MemoryScope.RUN)` extracts `entry.value`
- `_store_memory()`: `await binding.connector.write("latest", payload, MemoryScope.RUN)`
- All `binding.context` references removed, scope from `binding.manifest.scope`
- Updated `live_scenarios/research_audit/bootstrap.py` and `tests/service/test_e2e_phase5.py` to use `MemoryScope`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed broken imports in live_scenarios and e2e tests**
- **Found during:** Task 2
- **Issue:** `ConnectorScope` was removed from `zeroth.memory` exports but still imported in `live_scenarios/research_audit/bootstrap.py` and `tests/service/test_e2e_phase5.py`
- **Fix:** Replaced `ConnectorScope` imports with `MemoryScope` from `governai.memory.models`
- **Files modified:** `live_scenarios/research_audit/bootstrap.py`, `tests/service/test_e2e_phase5.py`
- **Commit:** ff63728

## Test Results

- `tests/memory/test_connectors.py`: 12 tests passing (protocol compliance, read/write/delete/search for all 3 connectors)
- `tests/memory/test_resolver.py`: 4 tests passing (Scoped+Auditing wrapping, no-emitter path, binding shape, MemoryScope)
- `tests/agent_runtime/`: 16 tests passing (all existing runner tests unaffected)
- Ruff lint: clean

## Decisions Made

1. **Storage layout**: Followed GovernAI's `DictMemoryConnector` pattern exactly (`_store[scope.value][target][key]`) for consistency
2. **Wrapping order**: AuditingMemoryConnector wraps first, ScopedMemoryConnector wraps second -- audit sees the raw scope values, scoped connector auto-resolves targets
3. **AgentRunner scope**: Uses `MemoryScope.RUN` for all operations since `ScopedMemoryConnector` handles target resolution based on run_id/thread_id

## Known Stubs

None -- all connectors are fully functional in-memory implementations.

## Self-Check: PASSED

- All 8 key files verified present
- All 3 commits verified (f6bd2f1, 27cd356, ff63728)
- 32 tests passing across memory and agent_runtime suites
