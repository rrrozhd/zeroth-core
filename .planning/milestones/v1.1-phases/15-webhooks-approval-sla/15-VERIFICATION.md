---
phase: 15-webhooks-approval-sla
verified: 2026-04-07T16:30:00Z
status: passed
score: 3/3 must-haves verified
re_verification: false
---

# Phase 15: Webhooks and Approval SLA Verification Report

**Phase Goal:** Callers receive durable push notifications on run completion, approval requests, and failure events, and approval SLA timeouts trigger escalation rather than silent expiry.
**Verified:** 2026-04-07T16:30:00Z
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A subscriber that registers a webhook URL receives an HTTP POST within a reasonable window after run completion or failure, even if the first delivery attempt fails | VERIFIED | WebhookService.emit_event enqueues deliveries per matching subscription; WebhookDeliveryWorker sends HTTP POST with HMAC-SHA256 signature; retry on failure with exponential backoff + jitter; 87 tests passing |
| 2 | A failed webhook delivery is retried with exponential backoff and eventually written to a dead-letter store rather than silently dropped | VERIFIED | delivery.py _handle_failure retries with next_retry_delay (jittered exp backoff), dead-letters after max_attempts exhausted; dead-letter replay available via REST API; tested in test_max_retries_exhausted_dead_letters, test_500_response_calls_mark_failed, test_timeout_calls_mark_failed |
| 3 | An approval that is not actioned within its configured SLA window escalates to the configured delegate or raises an alert rather than hanging indefinitely | VERIFIED | ApprovalSLAChecker polls list_overdue, calls ApprovalService.escalate with delegate/auto_reject/alert actions; ESCALATED status prevents double-escalation; tested in test_delegate_creates_new_record, test_auto_reject_resolves_as_rejected, test_alert_marks_escalated, test_already_escalated_is_noop |

**Score:** 3/3 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/zeroth/webhooks/models.py` | Webhook enums, subscription, delivery, dead-letter, payload models | VERIFIED | 134 lines, 7 classes: WebhookEventType, DeliveryStatus, EscalationAction, WebhookSubscription, WebhookDelivery, WebhookDeadLetter, WebhookEventPayload |
| `src/zeroth/webhooks/signing.py` | HMAC-SHA256 signing utility | VERIFIED | 22 lines, exports sign_payload |
| `src/zeroth/webhooks/repository.py` | Async repository for subscriptions, deliveries, dead-letters | VERIFIED | 374 lines, WebhookRepository class with full CRUD |
| `src/zeroth/webhooks/service.py` | WebhookService with emit_event, subscription management | VERIFIED | 109 lines, WebhookService class, uses repository |
| `src/zeroth/webhooks/delivery.py` | WebhookDeliveryWorker background poll-loop task | VERIFIED | 141 lines, poll_loop, _deliver with HMAC, _handle_failure with retry/dead-letter |
| `src/zeroth/service/webhook_api.py` | REST endpoints for webhook CRUD and dead-letter management | VERIFIED | 241 lines, register_webhook_routes, 6 endpoints |
| `src/zeroth/approvals/sla_checker.py` | ApprovalSLAChecker background poll-loop task | VERIFIED | 75 lines, polls list_overdue, escalates, emits webhook events |
| `src/zeroth/approvals/service.py` | Extended ApprovalService with escalation | VERIFIED | 482 lines, escalate method with delegate/auto_reject/alert, webhook emission on approval.requested and approval.resolved |
| `src/zeroth/approvals/repository.py` | Extended ApprovalRepository with list_overdue | VERIFIED | 176 lines, list_overdue query present |
| `src/zeroth/config/settings.py` | WebhookSettings and ApprovalSLASettings | VERIFIED | Both classes present |
| `src/zeroth/approvals/models.py` | ESCALATED status, SLA fields on ApprovalRecord | VERIFIED | ApprovalStatus.ESCALATED, sla_deadline, escalation_action fields |
| `src/zeroth/graph/models.py` | SLA config fields on HumanApprovalNodeData | VERIFIED | sla_timeout_seconds field present |
| `src/zeroth/migrations/versions/003_add_webhooks_and_sla.py` | Alembic migration for webhook tables and SLA columns | VERIFIED | 102 lines, revision 003 |
| `src/zeroth/service/bootstrap.py` | Full wiring for webhook and SLA subsystems | VERIFIED | Constructs WebhookRepository, WebhookService, WebhookDeliveryWorker, ApprovalSLAChecker |
| `src/zeroth/service/app.py` | Lifespan wiring for delivery worker and SLA checker | VERIFIED | Both delivery_worker and sla_checker started as asyncio tasks with graceful shutdown |
| `src/zeroth/orchestrator/runtime.py` | Webhook emission on run completion/failure | VERIFIED | _emit_webhook called for run.completed and run.failed events |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| webhooks/repository.py | storage/database.py | AsyncDatabase protocol | WIRED | Import and constructor injection confirmed |
| webhooks/service.py | webhooks/repository.py | self.repository | WIRED | Repository methods called in emit_event, create/list/deactivate |
| webhooks/delivery.py | webhooks/signing.py | sign_payload import | WIRED | Imported and used for X-Zeroth-Signature header |
| webhooks/delivery.py | webhooks/repository.py | claim/mark/dead_letter | WIRED | claim_pending_delivery, mark_delivered, mark_failed, dead_letter all called |
| service/app.py | webhooks/delivery.py | lifespan starts delivery worker | WIRED | poll_loop started as asyncio task, cancelled on shutdown |
| service/app.py | approvals/sla_checker.py | lifespan starts SLA checker | WIRED | poll_loop started as asyncio task, cancelled on shutdown |
| approvals/sla_checker.py | approvals/service.py | list_overdue and escalate | WIRED | repository.list_overdue and approval_service.escalate called |
| approvals/sla_checker.py | webhooks/service.py | emit_event for escalation | WIRED | Optional webhook_service.emit_event called after escalation |
| orchestrator/runtime.py | webhooks/service.py | emit_event after run | WIRED | _emit_webhook calls ws.emit_event for run.completed and run.failed |
| approvals/service.py | webhooks/service.py | emit_event on approval lifecycle | WIRED | emit_event called for approval.requested and approval.resolved |
| service/bootstrap.py | webhooks/service.py | Constructs WebhookService | WIRED | WebhookService, DeliveryWorker, SLAChecker all constructed |

### Data-Flow Trace (Level 4)

Not applicable -- this phase implements background workers and event-driven pipelines, not data-rendering components. Data flows verified through key link wiring and test coverage.

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| All 87 phase 15 tests pass | pytest (7 test files) | 87 passed in 1.18s | PASS |
| Webhook models importable | Verified via test execution | All model classes instantiated in tests | PASS |
| Delivery retry and dead-letter | Verified via test_max_retries_exhausted_dead_letters | Dead-lettering confirmed after max attempts | PASS |
| SLA escalation (all 3 actions) | Verified via test_delegate/auto_reject/alert | All three paths tested and passing | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| OPS-01 | 15-01, 15-02, 15-03 | Durable webhook notifications for run completion, approval needed, and failure events | SATISFIED | WebhookService emits events, DeliveryWorker sends HTTP POST with retry/dead-letter, REST API for subscription management, orchestrator and approval service emit all 5 event types |
| OPS-02 | 15-01, 15-03 | Approval SLA timeouts with escalation and delegation policies | SATISFIED | ApprovalSLAChecker polls for overdue approvals, ApprovalService.escalate supports delegate/auto_reject/alert, ESCALATED status prevents double-escalation, SLA deadline and escalation config on models |

No orphaned requirements found. REQUIREMENTS.md maps only OPS-01 and OPS-02 to Phase 15, both accounted for.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| webhooks/repository.py | 4 | Comment contains "placeholder" | Info | Refers to the SQL pattern borrowed from ApprovalRepository, not actual placeholder code |

No blockers or warnings found. No TODO/FIXME/PLACEHOLDER patterns in implementation code. No empty implementations, no stub returns.

### Human Verification Required

### 1. End-to-End Webhook Delivery

**Test:** Register a webhook subscription via POST /webhooks/subscriptions, trigger a run completion, verify the target URL receives an HTTP POST with correct payload and HMAC signature.
**Expected:** Target receives POST within poll_interval seconds with X-Zeroth-Signature header that validates against the subscription secret.
**Why human:** Requires running the full application with a real HTTP endpoint to receive callbacks.

### 2. Retry and Dead-Letter Under Real Network Conditions

**Test:** Register a webhook subscription pointing to an endpoint that returns 500 for the first N-1 attempts, then 200.
**Expected:** Delivery succeeds after retries with increasing delays. If the endpoint never recovers, the delivery appears in the dead-letters list.
**Why human:** Requires time-based observation of retry delays and a controllable mock endpoint.

### 3. SLA Escalation Timing

**Test:** Create an approval with a short SLA timeout (e.g., 5 seconds), wait without actioning it.
**Expected:** After the SLA deadline passes and the SLA checker polls, the approval transitions to ESCALATED and a new delegate approval is created (if delegate action configured).
**Why human:** Requires running the application with background tasks and observing time-based behavior.

### Gaps Summary

No gaps found. All three success criteria are fully implemented with real logic (no stubs), properly wired through the service/bootstrap/lifespan layers, and validated by 87 passing tests covering models, repository, service, delivery worker, REST API, SLA checker, escalation actions, and webhook event emission from both the orchestrator and approval service.

---

_Verified: 2026-04-07T16:30:00Z_
_Verifier: Claude (gsd-verifier)_
