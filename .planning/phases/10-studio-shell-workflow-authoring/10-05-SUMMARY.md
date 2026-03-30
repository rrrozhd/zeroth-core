---
phase: 10-studio-shell-workflow-authoring
plan: 05
subsystem: api
tags: [fastapi, studio, validation, contracts, sqlite, workflows]
requires:
  - phase: 10-02
    provides: "Dedicated Studio app/bootstrap, workflow routes, and scoped lease routes"
provides:
  - "Lease-protected draft save route backed by workflow scope and revision fencing"
  - "Workspace-scoped validation endpoint returning GraphValidationReport payloads"
  - "Slash-safe contract lookup route for refs like contract://input"
affects: [phase-10-frontend-authoring, studio-shell, autosave, node-contract-authoring]
tech-stack:
  added: []
  patterns: [workspace-scoped authoring APIs, lease and revision fencing, persisted-draft validation]
key-files:
  created:
    - src/zeroth/studio/validation_api.py
  modified:
    - src/zeroth/studio/workflows/service.py
    - src/zeroth/studio/workflows_api.py
    - src/zeroth/studio/app.py
    - tests/studio/test_validation_api.py
    - tests/studio/test_studio_app.py
key-decisions:
  - "Draft writes stay service-mediated so scope, active lease token, and revision token checks are enforced before graph persistence."
  - "Validation runs against the persisted workspace-scoped draft loaded from WorkflowService rather than any client-submitted graph payload."
  - "Workflow detail responses include authoritative last_saved_at timestamps so authoring clients can reflect backend save state."
patterns-established:
  - "Studio write routes resolve tenant/workspace from the authenticated principal and never trust client-provided scope."
  - "Node-local contract lookup uses FastAPI path converters with backend ContractRegistry resolution for slash-containing refs."
requirements-completed: [STU-02, AST-04]
duration: 3min
completed: 2026-03-30
---

# Phase 10 Plan 05: Authoring Validation Summary

**Lease-fenced draft saves, persisted graph validation, and slash-safe contract schema lookup for Studio authoring flows**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-30T17:09:26+03:00
- **Completed:** 2026-03-30T17:12:02+03:00
- **Tasks:** 1
- **Files modified:** 10

## Accomplishments
- Added `PUT /studio/workflows/{workflow_id}/draft` with workspace scope enforcement, active lease validation, stale revision rejection, revision rotation, and `last_saved_at` updates.
- Added `POST /studio/workflows/{workflow_id}/validate` to return the exact `GraphValidationReport` for the currently persisted draft graph inside the caller's scope.
- Added `GET /studio/contracts/{contract_ref:path}` so node-local authoring flows can resolve real refs such as `contract://input`.

## Task Commits

Each task was committed atomically:

1. **Task 1: Add scope-aware draft save and validation and slash-safe contract lookup APIs** - `23b2864` (test), `11bd0e2` (feat)

## Files Created/Modified
- `src/zeroth/studio/validation_api.py` - Studio validation and contract lookup routes.
- `src/zeroth/studio/workflows/service.py` - scoped draft update logic with lease and revision fencing.
- `src/zeroth/studio/workflows_api.py` - draft save request/response contract and HTTP error mapping.
- `src/zeroth/studio/app.py` - validation route registration.
- `src/zeroth/studio/workflows/repository.py` - draft-head metadata updates after persisted saves.
- `src/zeroth/studio/leases/repository.py` - active lease lookup used by draft-save fencing.
- `tests/studio/test_validation_api.py` - coverage for successful save, conflicts, validation payloads, and slash-safe contract lookup.
- `tests/studio/test_studio_app.py` - workflow detail regression coverage for `last_saved_at`.

## Decisions Made
- Returned `last_saved_at` from workflow detail/save responses so the backend remains the source of truth for save state.
- Kept validation backend-authoritative by loading the persisted draft through `WorkflowService` before calling `GraphValidator`.
- Mapped missing or stale lease/revision conditions to explicit `409` responses while foreign-scope callers still receive `404`.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated adjacent Studio API regression coverage for the expanded detail payload**
- **Found during:** Task 1
- **Issue:** Adding `last_saved_at` to the authoring-safe workflow detail response broke the existing Studio app route assertion.
- **Fix:** Updated `tests/studio/test_studio_app.py` to assert the authoritative `last_saved_at` field.
- **Files modified:** `tests/studio/test_studio_app.py`
- **Verification:** `uv run pytest tests/studio/test_validation_api.py tests/studio/test_studio_app.py -q`
- **Committed in:** `11bd0e2`

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** The deviation was a direct regression fix caused by the new response contract. No scope creep.

## Issues Encountered

- The initial validation expectation only asserted one issue, but the existing validator correctly returns both `empty_graph` and `missing_entrypoint` for an empty draft. The test was tightened to the real normalized payload.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Backend authoring routes now expose the save, validate, and contract lookup contracts the frontend autosave and node-inspector work can depend on.
- The next Studio plan can treat lease token, revision token, `last_saved_at`, `GraphValidationReport`, and slash-safe contract refs as stable backend surfaces.

## Self-Check

PASSED
