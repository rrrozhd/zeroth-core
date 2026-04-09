---
phase: 14-memory-connectors-container-sandbox
plan: 04
subsystem: sandbox
tags: [docker, sidecar, fastapi, httpx, network-isolation, container-security]

requires:
  - phase: 08-runtime-security
    provides: "SandboxManager with LOCAL/DOCKER backends, ResourceConstraints, build_docker_resource_flags"
  - phase: 11-config-postgres-storage
    provides: "ZerothSettings unified config model"
provides:
  - "SidecarExecutor with per-execution Docker network isolation"
  - "FastAPI sidecar REST API (execute/status/cancel/health)"
  - "SandboxSidecarClient HTTP client for API container"
  - "SandboxBackendMode.SIDECAR enum value and _run_via_sidecar dispatch"
  - "SandboxSettings in ZerothSettings"
affects: [deployment, docker-compose, sandbox-hardening]

tech-stack:
  added: [fastapi-sidecar, httpx-async-client, docker-network-isolation]
  patterns: [sidecar-architecture, sync-async-bridge-via-asyncio-run]

key-files:
  created:
    - src/zeroth/sandbox_sidecar/__init__.py
    - src/zeroth/sandbox_sidecar/models.py
    - src/zeroth/sandbox_sidecar/executor.py
    - src/zeroth/sandbox_sidecar/app.py
    - src/zeroth/execution_units/sidecar_client.py
    - tests/execution_units/test_sidecar_client.py
    - tests/sandbox_sidecar/__init__.py
    - tests/sandbox_sidecar/test_app.py
  modified:
    - src/zeroth/execution_units/sandbox.py
    - src/zeroth/config/settings.py
    - tests/execution_units/test_sandbox.py

key-decisions:
  - "asyncio.run() bridge in _run_via_sidecar since SandboxManager.run() is sync and no callers are async yet"
  - "Sidecar creates --internal Docker network per execution for untrusted code isolation"
  - "SandboxSidecarClient uses httpx.AsyncClient for non-blocking HTTP calls"

patterns-established:
  - "Sidecar pattern: API container delegates Docker operations to a separate service over HTTP"
  - "Per-execution network isolation: each sandbox run gets its own Docker network"

requirements-completed: [SBX-01, SBX-02]

duration: 6min
completed: 2026-04-07
---

# Phase 14 Plan 04: Container Sandbox Sidecar Summary

**Sandbox sidecar architecture with per-execution Docker network isolation, FastAPI REST API, httpx client, and SIDECAR backend mode in SandboxManager**

## Performance

- **Duration:** 6 min (371s)
- **Started:** 2026-04-07T08:24:01Z
- **Completed:** 2026-04-07T08:30:12Z
- **Tasks:** 2
- **Files modified:** 11

## Accomplishments
- Sandbox sidecar FastAPI service with execute/status/cancel/health REST endpoints
- SandboxSidecarClient HTTP client using httpx.AsyncClient for API container communication
- SandboxManager SIDECAR backend mode that dispatches through client (API container never touches Docker socket)
- Per-execution Docker network isolation with --internal flag for untrusted code
- Resource limits applied via existing build_docker_resource_flags
- SandboxSettings added to ZerothSettings unified config
- 23 total new tests (13 sidecar + client, 5 SIDECAR mode, 5 existing tests continue passing)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create sidecar service package and HTTP client** - `d4edf02` (feat)
2. **Task 2: Integrate SIDECAR mode into SandboxManager and config** - `4b8b25a` (feat)

## Files Created/Modified
- `src/zeroth/sandbox_sidecar/models.py` - Pydantic request/response schemas for sidecar API
- `src/zeroth/sandbox_sidecar/executor.py` - Docker execution with per-execution --internal network isolation
- `src/zeroth/sandbox_sidecar/app.py` - FastAPI sidecar application (execute/status/cancel/health)
- `src/zeroth/sandbox_sidecar/__init__.py` - Package init exporting app
- `src/zeroth/execution_units/sidecar_client.py` - httpx.AsyncClient HTTP client for sidecar
- `src/zeroth/execution_units/sandbox.py` - Added SIDECAR enum, sidecar_url, _run_via_sidecar dispatch
- `src/zeroth/config/settings.py` - Added SandboxSettings to ZerothSettings
- `tests/execution_units/test_sidecar_client.py` - 6 tests for HTTP client
- `tests/sandbox_sidecar/test_app.py` - 7 tests for FastAPI endpoints
- `tests/execution_units/test_sandbox.py` - 5 new tests for SIDECAR mode

## Decisions Made
- Used asyncio.run() to bridge sync SandboxManager.run() to async sidecar client (no callers are async yet)
- Sidecar creates --internal Docker network per execution for untrusted code isolation
- SandboxSidecarClient uses httpx.AsyncClient for non-blocking HTTP calls
- Fixed returncode mapping: `response.returncode if response.returncode is not None else 1` (not `or 1` which breaks on 0)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed returncode mapping in _run_via_sidecar**
- **Found during:** Task 2 (SIDECAR mode integration)
- **Issue:** Plan specified `response.returncode or 1` which evaluates to 1 when returncode is 0 (falsy)
- **Fix:** Changed to `response.returncode if response.returncode is not None else 1`
- **Files modified:** src/zeroth/execution_units/sandbox.py
- **Verification:** test_run_via_sidecar_constructs_request_and_translates_response passes with returncode=0
- **Committed in:** 4b8b25a (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug fix)
**Impact on plan:** Essential correctness fix. No scope creep.

## Issues Encountered
None beyond the auto-fixed returncode bug.

## Known Stubs
None - all data paths are fully wired.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Sidecar architecture complete, ready for docker-compose deployment wiring
- SandboxManager can dispatch to LOCAL, DOCKER, AUTO, or SIDECAR backends
- Config supports backend selection via zeroth.yaml or environment variables

---
*Phase: 14-memory-connectors-container-sandbox*
*Completed: 2026-04-07*
