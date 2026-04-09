---
phase: 15-webhooks-approval-sla
plan: 03
subsystem: approvals, orchestrator, webhooks
tags: [sla, escalation, webhooks, event-emission, background-tasks]

requires:
  - phase: 15-01
    provides: webhook models, signing, config settings, SLA fields on ApprovalRecord
  - phase: 15-02
    provides: WebhookService, WebhookDeliveryWorker, WebhookRepository, webhook REST API

provides:
  - ApprovalSLAChecker background poll loop for overdue approval detection and escalation
  - ApprovalService.escalate with delegate/auto_reject/alert actions
  - ApprovalRepository.list_overdue query
  - Webhook event emission from orchestrator (run.completed, run.failed) and approval service (approval.requested, approval.resolved)
  - Full ServiceBootstrap wiring for all Phase 15 components
  - SLA checker integrated into app lifespan

affects: [deployment, operations, monitoring]

tech-stack:
  added: []
  patterns: [background-poll-loop, optional-service-injection, event-emission-hooks]

key-files:
  created:
    - src/zeroth/approvals/sla_checker.py
    - tests/test_approval_sla.py
    - tests/test_webhook_event_emission.py
  modified:
    - src/zeroth/approvals/repository.py
    - src/zeroth/approvals/service.py
    - src/zeroth/orchestrator/runtime.py
    - src/zeroth/service/bootstrap.py
    - src/zeroth/service/app.py

key-decisions:
  - "SLA checker uses optional WebhookService injection to avoid circular imports between approvals and webhooks"
  - "Webhook emission uses fire-and-forget with exception logging (never blocks main flow)"
  - "ApprovalRepository.write now persists sla_deadline, escalation_action, escalated_from_id as indexed columns"

patterns-established:
  - "Optional service injection via object | None for cross-subsystem dependencies"
  - "Background poll loop pattern consistent with RunWorker and WebhookDeliveryWorker"

requirements-completed: [OPS-02, OPS-01]

duration: 14min
completed: 2026-04-07
---

# Phase 15 Plan 03: Approval SLA Enforcement and Webhook Event Emission Summary

**Approval SLA checker with delegate/auto-reject/alert escalation, webhook event hooks on all run and approval lifecycle events, full ServiceBootstrap wiring**

## Performance

- **Duration:** 14 min (858s)
- **Started:** 2026-04-07T13:01:25Z
- **Completed:** 2026-04-07T13:15:43Z
- **Tasks:** 2
- **Files modified:** 9

## Accomplishments
- ApprovalSLAChecker polls for overdue approvals and escalates via configured action (delegate creates new approval, auto_reject resolves as rejected, alert marks escalated)
- Double-escalation prevented by ESCALATED status check (no-op on already-escalated approvals)
- Webhook events emitted on all 5 event types: run.completed, run.failed, approval.requested, approval.resolved, approval.escalated
- ServiceBootstrap fully wires WebhookRepository, WebhookService, WebhookDeliveryWorker, and ApprovalSLAChecker
- SLA checker runs as background task in app lifespan with graceful shutdown

## Task Commits

Each task was committed atomically:

1. **Task 1: ApprovalSLAChecker, escalation, overdue query** (TDD)
   - `ddb50cd` (test) - failing tests for approval SLA enforcement
   - `ee39f92` (feat) - implement approval SLA checker, escalation, and overdue query
2. **Task 2: Event emission hooks and ServiceBootstrap wiring** - `7223522` (feat)

## Files Created/Modified
- `src/zeroth/approvals/sla_checker.py` - Background poll loop for overdue approval detection and escalation
- `src/zeroth/approvals/repository.py` - Added list_overdue query and SLA column persistence in write
- `src/zeroth/approvals/service.py` - Added escalate method, SLA-aware create_pending, webhook emission
- `src/zeroth/orchestrator/runtime.py` - Webhook event emission on run completion/failure and approval creation
- `src/zeroth/service/bootstrap.py` - Full wiring for WebhookRepository, WebhookService, DeliveryWorker, SLAChecker
- `src/zeroth/service/app.py` - SLA checker task in lifespan with graceful shutdown
- `tests/test_approval_sla.py` - 14 tests for SLA enforcement
- `tests/test_webhook_event_emission.py` - 6 tests for webhook event emission

## Decisions Made
- Used optional `object | None` typing for webhook_service to avoid circular imports between approvals and webhooks packages
- Webhook emission is fire-and-forget (exception logged, never blocks main flow)
- ApprovalRepository.write extended to persist sla_deadline/escalation_action/escalated_from_id as indexed SQL columns for efficient list_overdue queries

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] ActorIdentity API mismatch in plan code**
- **Found during:** Task 1 (tests)
- **Issue:** Plan specified ActorIdentity with identity_type/identifier/display_name fields which don't exist; actual API uses subject/auth_method
- **Fix:** Used correct ActorIdentity constructor fields throughout tests and escalate method
- **Files modified:** tests/test_approval_sla.py, src/zeroth/approvals/service.py
- **Committed in:** ee39f92 (Task 1 commit)

**2. [Rule 2 - Missing Critical] ApprovalRepository.write not persisting SLA columns**
- **Found during:** Task 1 (implementation)
- **Issue:** The write method only persisted core columns but not sla_deadline/escalation_action/escalated_from_id, making list_overdue query ineffective
- **Fix:** Extended INSERT/UPDATE statement to include all three SLA columns
- **Files modified:** src/zeroth/approvals/repository.py
- **Committed in:** ee39f92 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (1 bug, 1 missing critical)
**Impact on plan:** Both fixes necessary for correctness. No scope creep.

## Issues Encountered
- Pre-existing test failures in test_checkpoints_do_not_persist_raw_secret_values and test_phase5_thread_continuity unrelated to this plan's changes
- Ruff linting required logger placement after all imports to avoid E402 errors

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Phase 15 is now complete: webhook models, signing, config, delivery, REST API, SLA checker, and event emission all wired
- Ready for phase transition and verification

## Known Stubs
None - all functionality is fully wired with real implementations.

---
*Phase: 15-webhooks-approval-sla*
*Completed: 2026-04-07*
