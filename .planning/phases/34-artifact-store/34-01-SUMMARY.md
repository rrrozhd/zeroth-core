---
phase: 34-artifact-store
plan: 01
subsystem: storage
tags: [artifacts, redis, filesystem, protocol, pydantic, asyncio]

# Dependency graph
requires: []
provides:
  - "zeroth.core.artifacts package with ArtifactStore Protocol"
  - "ArtifactReference Pydantic model for lightweight artifact pointers"
  - "RedisArtifactStore with pipeline-atomic SETEX and scan_iter cleanup"
  - "FilesystemArtifactStore with .meta.json sidecars and asyncio.to_thread"
  - "ArtifactStoreSettings wired into ZerothSettings"
  - "Error hierarchy: ArtifactStoreError, ArtifactNotFoundError, ArtifactStorageError, ArtifactTTLError"
  - "generate_artifact_key for hierarchical key generation"
affects: [artifact-store, orchestrator, audit]

# Tech tracking
tech-stack:
  added: []
  patterns: [protocol-based storage interface, sidecar metadata files, lazy TTL expiration, pipeline-atomic writes]

key-files:
  created:
    - src/zeroth/core/artifacts/__init__.py
    - src/zeroth/core/artifacts/models.py
    - src/zeroth/core/artifacts/errors.py
    - src/zeroth/core/artifacts/store.py
    - tests/artifacts/__init__.py
    - tests/artifacts/test_models.py
    - tests/artifacts/test_store.py
  modified:
    - src/zeroth/core/config/settings.py

key-decisions:
  - "RedisArtifactStore constructor accepts optional client parameter for testability without redis dependency"
  - "FilesystemArtifactStore uses lazy TTL expiration on retrieve rather than background cleanup"
  - "Path traversal prevention by rejecting keys with '..' path segments"

patterns-established:
  - "Protocol-based storage interface: ArtifactStore Protocol with 6 async methods following ThreadStateStore pattern"
  - "Sidecar metadata: .meta.json companion files for filesystem artifact metadata"
  - "Lazy expiration: check and cleanup expired artifacts at read time"

requirements-completed: [ARTF-01, ARTF-02, ARTF-03]

# Metrics
duration: 10min
completed: 2026-04-12
---

# Phase 34 Plan 01: Artifact Store Core Package Summary

**ArtifactStore Protocol with Redis pipeline-atomic and filesystem sidecar backends, ArtifactReference model, error hierarchy, settings integration, and 48 unit tests**

## Performance

- **Duration:** 10 min
- **Started:** 2026-04-12T21:46:30Z
- **Completed:** 2026-04-12T21:56:39Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments
- Created zeroth.core.artifacts package with complete ArtifactStore Protocol (6 async methods)
- Implemented RedisArtifactStore with pipeline-atomic SETEX for data+metadata and scan_iter for prefix-based cleanup
- Implemented FilesystemArtifactStore with .meta.json sidecars, asyncio.to_thread wrapping, and lazy TTL expiration
- ArtifactReference Pydantic model with JSON round-trip, extra=forbid, UTC timestamps
- ArtifactStoreSettings wired into ZerothSettings with filesystem/redis backend config
- Error hierarchy following contracts/errors.py pattern
- Hierarchical key generation ({run_id}/{node_id}/{uuid4_hex}) for prefix-based bulk cleanup
- Path traversal prevention (T-34-01) and max size enforcement (T-34-02)
- 48 unit tests all passing, lint and format clean

## Task Commits

Each task was committed atomically (TDD: test then feat):

1. **Task 1: ArtifactReference model, errors, settings, key generation**
   - `add6cd5` (test) - Failing tests for model, errors, settings, key gen
   - `cde5139` (feat) - Implementation passing all 19 tests
2. **Task 2: ArtifactStore Protocol and Redis/Filesystem implementations**
   - `9bae725` (test) - Failing tests for Protocol and both backends
   - `cd7804c` (feat) - Implementation passing all 29 tests

## Files Created/Modified
- `src/zeroth/core/artifacts/__init__.py` - Barrel exports for all public symbols
- `src/zeroth/core/artifacts/models.py` - ArtifactReference model, ArtifactStoreSettings, generate_artifact_key
- `src/zeroth/core/artifacts/errors.py` - Error hierarchy (base + 3 specific errors)
- `src/zeroth/core/artifacts/store.py` - ArtifactStore Protocol, RedisArtifactStore, FilesystemArtifactStore
- `src/zeroth/core/config/settings.py` - Added artifact_store field to ZerothSettings
- `tests/artifacts/__init__.py` - Test package init
- `tests/artifacts/test_models.py` - 19 tests for models, errors, settings, key gen, exports
- `tests/artifacts/test_store.py` - 29 tests for Protocol, Redis backend (mocked), filesystem backend (tmp_path)

## Decisions Made
- RedisArtifactStore constructor accepts optional `client` keyword parameter to inject a mock Redis client for testing, avoiding the need for the `redis` package at test time
- FilesystemArtifactStore implements lazy TTL expiration -- checks and cleans up expired artifacts at retrieve time rather than running a background sweep process
- Path traversal prevention validates key segments for ".." rather than resolving absolute paths, keeping validation simple and explicit

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added client injection to RedisArtifactStore constructor**
- **Found during:** Task 2 (Redis implementation)
- **Issue:** The `redis` Python package is an optional dependency not installed in test environment. Constructor's eager `import redis.asyncio` caused ModuleNotFoundError even when tests planned to mock the client.
- **Fix:** Added optional `client` keyword parameter to constructor. When provided, skips redis import and uses injected client directly.
- **Files modified:** src/zeroth/core/artifacts/store.py
- **Verification:** All 29 store tests pass with mocked client
- **Committed in:** cd7804c (Task 2 commit)

**2. [Rule 3 - Blocking] Fixed test mock setup for Redis pipeline async context manager**
- **Found during:** Task 2 (Redis test execution)
- **Issue:** AsyncMock for `client.pipeline()` returned a coroutine instead of an async context manager. Real `redis.asyncio` `pipeline()` is synchronous, returning an async CM.
- **Fix:** Changed mock_redis fixture from AsyncMock to MagicMock with explicitly configured async methods (get, exists, delete as AsyncMock; pipeline as MagicMock returning async CM).
- **Files modified:** tests/artifacts/test_store.py
- **Verification:** All 12 Redis tests pass
- **Committed in:** cd7804c (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (2 blocking)
**Impact on plan:** Both auto-fixes necessary for test execution. No scope creep. Production code behavior unchanged.

## Issues Encountered
None beyond the auto-fixed deviations above.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Artifact store package is self-contained and ready for integration
- Future phases can import from zeroth.core.artifacts for artifact externalization
- Redis backend requires redis package as optional dependency (already in project's optional deps)
- Filesystem backend works with zero external dependencies

## Self-Check: PASSED

All 8 created files verified on disk. All 4 task commits (add6cd5, cde5139, 9bae725, cd7804c) verified in git log. No stubs detected. No untracked threat surface found.

---
*Phase: 34-artifact-store*
*Completed: 2026-04-12*
