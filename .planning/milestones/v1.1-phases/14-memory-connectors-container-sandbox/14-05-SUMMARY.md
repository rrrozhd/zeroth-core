---
phase: 14-memory-connectors-container-sandbox
plan: 05
subsystem: memory
tags: [redis, pgvector, chromadb, elasticsearch, factory-pattern, registry, bootstrap]

requires:
  - phase: 14-01
    provides: "InMemoryConnectorRegistry, MemoryConnectorResolver, ConnectorManifest, in-memory connectors"
  - phase: 14-02
    provides: "RedisKVMemoryConnector, RedisThreadMemoryConnector"
  - phase: 14-03
    provides: "PgvectorMemoryConnector, ChromaDBMemoryConnector, ElasticsearchMemoryConnector"
provides:
  - "register_memory_connectors() factory that creates and registers all configured connectors"
  - "memory_registry field on ServiceBootstrap populated at startup"
  - "Agents can resolve connectors by type name string at runtime"
affects: [agent-runtime, service-bootstrap, memory-system]

tech-stack:
  added: []
  patterns: ["Lazy import with contextlib.suppress for optional backends", "Settings-driven factory registration pattern", "Singleton connector instances via registry"]

key-files:
  created: [src/zeroth/memory/factory.py, tests/memory/test_factory.py]
  modified: [src/zeroth/memory/__init__.py, src/zeroth/service/bootstrap.py]

key-decisions:
  - "Used duck-typed settings (Any) instead of concrete ZerothSettings to avoid blocking on config module creation"
  - "_BootstrapMemorySettings helper in bootstrap.py provides default shape until ZerothSettings exists"
  - "Lazy imports with contextlib.suppress for all optional connector modules (redis_kv, pgvector, chroma, elastic)"

patterns-established:
  - "Factory registration: register_memory_connectors() creates singletons and populates registry"
  - "Optional backend pattern: contextlib.suppress(ImportError) for modules created by parallel agents"

requirements-completed: [MEM-01, MEM-02, MEM-03, MEM-04, MEM-05]

duration: 224s
completed: 2026-04-07
---

# Phase 14 Plan 05: Memory Connector Factory and Bootstrap Wiring Summary

**Factory function registers all 8 memory connector types (3 in-memory + 2 Redis + 3 vector/search) in InMemoryConnectorRegistry at ServiceBootstrap startup**

## Performance

- **Duration:** 224s (3m 44s)
- **Started:** 2026-04-07T08:48:14Z
- **Completed:** 2026-04-07T08:52:18Z
- **Tasks:** 1 (TDD: RED + GREEN)
- **Files modified:** 4

## Accomplishments
- Created `register_memory_connectors()` factory that reads settings and creates+registers all enabled connector instances
- Wired factory into ServiceBootstrap so memory_registry is populated at service startup
- All external backends (Redis, pgvector, ChromaDB, Elasticsearch) conditionally registered based on settings
- In-memory connectors (ephemeral, key_value, thread) always registered for dev/test
- 21 unit tests covering all registration paths, config propagation, and singleton behavior

## Task Commits

Each task was committed atomically:

1. **Task 1 (RED): Failing tests for connector factory** - `4d87f41` (test)
2. **Task 1 (GREEN): Implement factory and bootstrap wiring** - `a385caf` (feat)

_TDD task: test commit followed by implementation commit_

## Files Created/Modified
- `src/zeroth/memory/factory.py` - Factory with register_memory_connectors() and per-backend helpers
- `src/zeroth/memory/__init__.py` - Added register_memory_connectors to package exports
- `src/zeroth/service/bootstrap.py` - Added memory_registry field and registration call
- `tests/memory/test_factory.py` - 21 tests covering default, Redis, pgvector, ChromaDB, Elasticsearch registration

## Decisions Made
- Used duck-typed `Any` for settings parameter since ZerothSettings module does not exist yet (parallel execution)
- Created `_BootstrapMemorySettings` helper class in bootstrap.py to provide default settings shape
- Used `contextlib.suppress(ImportError)` for lazy imports of optional connector modules created by parallel agents
- Tests use `unittest.mock.patch` to mock external connector classes, avoiding dependency on uninstalled packages

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] No ZerothSettings class exists yet**
- **Found during:** Task 1 (implementation)
- **Issue:** Plan references `ZerothSettings` from `src/zeroth/config/settings.py` but that module does not exist (it was planned for Phase 11 but may not be in this worktree)
- **Fix:** Used duck-typed `Any` for settings parameter in factory; created `_BootstrapMemorySettings` in bootstrap.py with default values matching the plan's settings shape
- **Files modified:** `src/zeroth/memory/factory.py`, `src/zeroth/service/bootstrap.py`
- **Verification:** All 21 tests pass with mock settings objects
- **Committed in:** `a385caf`

**2. [Rule 3 - Blocking] Connector modules from Plans 02/03 not present in worktree**
- **Found during:** Task 1 (implementation)
- **Issue:** `redis_kv.py`, `redis_thread.py`, `pgvector_connector.py`, `chroma_connector.py`, `elastic_connector.py` are created by parallel agents and not yet in this worktree
- **Fix:** Used `contextlib.suppress(ImportError)` for all optional connector imports; tests mock the connector classes via `unittest.mock.patch`
- **Files modified:** `src/zeroth/memory/factory.py`
- **Verification:** All tests pass; factory gracefully handles missing modules
- **Committed in:** `a385caf`

---

**Total deviations:** 2 auto-fixed (2 blocking issues from parallel execution)
**Impact on plan:** Both fixes necessary for parallel agent compatibility. No scope creep. Factory will work correctly once connector modules from Plans 02/03 are merged.

## Issues Encountered
None beyond the deviations documented above.

## User Setup Required
None - no external service configuration required.

## Known Stubs
None - all registration logic is fully wired. External connector registration is conditional on settings, not stubbed.

## Next Phase Readiness
- Memory connector registry is populated at bootstrap, ready for agent runtime resolution
- When Plans 02/03 connector modules merge, the lazy imports will resolve and external backends will register
- When ZerothSettings is available, `_BootstrapMemorySettings` can be replaced with real settings

---
*Phase: 14-memory-connectors-container-sandbox*
*Completed: 2026-04-07*
