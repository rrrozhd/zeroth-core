---
phase: 10-studio-shell-workflow-authoring
plan: 01
subsystem: database
tags: [studio, sqlite, workflows, leases, graph-repository]
requires:
  - phase: 9-durable-control-plane
    provides: SQLite-backed persistence, tenant/workspace scope patterns, and graph lifecycle primitives
provides:
  - Workspace-owned workflow metadata persistence in Studio tables
  - Draft-head lookup that resolves the current mutable graph without deployments
  - Scoped workflow edit leases with conflict detection
affects: [phase-10-plan-02, phase-10-plan-05, studio-api, autosave]
tech-stack:
  added: []
  patterns: [Dedicated studio metadata tables, scoped service filtering, lease-backed draft editing]
key-files:
  created:
    - src/zeroth/studio/__init__.py
    - src/zeroth/studio/models.py
    - src/zeroth/studio/workflows/repository.py
    - src/zeroth/studio/workflows/service.py
    - src/zeroth/studio/leases/models.py
    - src/zeroth/studio/leases/repository.py
    - src/zeroth/studio/leases/service.py
    - tests/studio/test_workflows_repository.py
  modified:
    - PROGRESS.md
key-decisions:
  - "Studio workflow ownership lives in dedicated metadata tables while graph JSON stays only in graph_versions."
  - "Workflow and lease reads must always filter by tenant_id and workspace_id instead of relying on downstream route wiring."
patterns-established:
  - "WorkflowService composes WorkflowRepository with GraphRepository to keep authoring metadata separate from draft content."
  - "WorkflowLeaseService validates scope through WorkflowRepository before mutating lease rows."
requirements-completed: [STU-02]
duration: 3min
completed: 2026-03-30
---

# Phase 10 Plan 01: Studio workflow persistence foundations Summary

**Workspace-scoped Studio workflow metadata, draft-head lookup, and edit leases backed by SQLite without duplicating graph payloads**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-30T13:53:14Z
- **Completed:** 2026-03-30T13:56:39Z
- **Tasks:** 1
- **Files modified:** 11

## Accomplishments
- Added a new `zeroth.studio` package with strict workflow, draft-head, and lease models.
- Persisted workflow ownership and draft-head state in Studio-owned tables while leaving graph payloads in `graph_versions`.
- Enforced scoped workflow lookup and exclusive draft editing with lease acquire, renew, release, and conflict behavior proven by tests.

## Task Commits

Each task was committed atomically:

1. **Task 1: Create workspace-owned workflow metadata, draft-head, and lease persistence foundations** - `dbd6073` (test)
2. **Task 1: Create workspace-owned workflow metadata, draft-head, and lease persistence foundations** - `17155d8` (feat)

## Files Created/Modified
- `src/zeroth/studio/models.py` - strict Studio workflow and lease models shared across repositories and services
- `src/zeroth/studio/workflows/repository.py` - workflow metadata and draft-head migrations plus scoped query helpers
- `src/zeroth/studio/workflows/service.py` - workflow creation, list, and lookup composed with `GraphRepository`
- `src/zeroth/studio/leases/repository.py` - workflow lease storage with expiry and conflict semantics
- `src/zeroth/studio/leases/service.py` - scope-aware lease acquire, renew, and release operations
- `tests/studio/test_workflows_repository.py` - red/green coverage for schema fields, scope isolation, graph linkage, and lease conflicts

## Decisions Made
- Stored authoring ownership and current draft lineage in Studio tables instead of extending `graph_versions` with workspace metadata.
- Initialized the lease repository as part of Studio workflow service bootstrap so workflow creation leaves the full authoring persistence boundary ready for subsequent API wiring.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- The red test command initially failed with the expected `ModuleNotFoundError` because `zeroth.studio` did not exist yet; this was the intended TDD baseline and was resolved by implementing the package.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Phase `10-02` can build Studio bootstrap and API wiring on top of the new workflow and lease services.
- The persistence boundary now preserves draft-vs-runtime separation and workspace scoping needed for autosave and lease-aware APIs.

## Self-Check: PASSED

- Found `.planning/phases/10-studio-shell-workflow-authoring/10-01-SUMMARY.md`.
- Found `src/zeroth/studio/workflows/repository.py`, `src/zeroth/studio/leases/service.py`, and `tests/studio/test_workflows_repository.py`.
- Verified task commits `dbd6073` and `17155d8` exist in git history.
