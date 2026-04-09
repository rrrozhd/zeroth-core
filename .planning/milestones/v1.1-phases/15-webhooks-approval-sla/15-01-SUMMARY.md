---
phase: 15-webhooks-approval-sla
plan: 01
subsystem: webhooks
tags: [webhooks, approval-sla, models, repository, migration, signing]
dependency_graph:
  requires: []
  provides: [webhook-models, webhook-repository, webhook-signing, approval-sla-fields, migration-003]
  affects: [approvals, graph-models, config-settings]
tech_stack:
  added: [hmac-sha256-signing]
  patterns: [webhook-subscription, delivery-lifecycle, dead-letter-queue, exponential-backoff-jitter]
key_files:
  created:
    - src/zeroth/webhooks/__init__.py
    - src/zeroth/webhooks/models.py
    - src/zeroth/webhooks/signing.py
    - src/zeroth/webhooks/repository.py
    - src/zeroth/migrations/versions/003_add_webhooks_and_sla.py
    - tests/test_webhook_models.py
    - tests/test_webhook_repository.py
  modified:
    - src/zeroth/config/settings.py
    - src/zeroth/approvals/models.py
    - src/zeroth/graph/models.py
decisions:
  - Event types as JSON array string in SQLite for subscription storage
  - Exponential backoff with full jitter for retry scheduling
  - All SLA fields nullable for backward compatibility
  - EscalationAction stored as string in graph models to avoid import dependency
metrics:
  duration: 352s
  completed: "2026-04-07"
  tasks: 2
  files: 10
---

# Phase 15 Plan 01: Webhook and SLA Data Foundation Summary

Webhook models, HMAC-SHA256 signing, WebhookRepository with full delivery lifecycle, Alembic migration 003, and approval/graph SLA field extensions -- all with 38 passing tests.

## What Was Built

### Task 1: Webhook models, signing, config, and SLA extensions

Created the `src/zeroth/webhooks/` package with:
- **WebhookEventType** enum: run.completed, run.failed, approval.requested, approval.resolved, approval.escalated
- **DeliveryStatus** enum: pending, delivered, failed, dead_letter
- **EscalationAction** enum: delegate, auto_reject, alert
- **WebhookSubscription**: auto-generates subscription_id and secret, tracks deployment_ref, target_url, event_types
- **WebhookDelivery**: tracks delivery lifecycle with attempt_count, max_attempts, exponential backoff scheduling
- **WebhookDeadLetter**: preserves failed delivery metadata for inspection
- **WebhookEventPayload**: standard event envelope with event_type, event_id, timestamp, data
- **sign_payload()**: HMAC-SHA256 signing utility matching `hmac.new(secret, payload, sha256).hexdigest()`

Extended existing models:
- **ApprovalStatus.ESCALATED** added (prevents double-escalation)
- **ApprovalRecord**: sla_deadline, escalation_action, escalated_from_id (all nullable)
- **HumanApprovalNodeData**: sla_timeout_seconds, escalation_action, delegate_identity (all nullable)
- **WebhookSettings** and **ApprovalSLASettings** added to ZerothSettings

### Task 2: Alembic migration 003 and WebhookRepository

- **Migration 003**: Creates webhook_subscriptions, webhook_deliveries, webhook_dead_letters tables with proper indexes. Adds sla_deadline, escalation_action, escalated_from_id nullable columns to approvals table.
- **WebhookRepository**: Full async CRUD following ApprovalRepository patterns:
  - Subscription: create, get, list, list_for_event, deactivate, delete
  - Delivery: enqueue, claim_pending (atomic SELECT+UPDATE), mark_delivered, mark_failed (with exponential backoff + jitter), dead_letter
  - Dead-letter: list (ordered DESC by dead_lettered_at), get

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 (RED) | 04121a0 | Failing tests for webhook models, signing, config, SLA extensions |
| 1 (GREEN) | da2cfa5 | Implement webhook models, signing, config, SLA extensions |
| 2 (RED) | 6c1b851 | Failing tests for WebhookRepository and migration 003 |
| 2 (GREEN) | c59a582 | Implement migration 003 and WebhookRepository |

## Verification

- 38 tests passing (19 model/signing + 19 repository/migration)
- Ruff lint clean on all new and modified files
- All new fields on existing models are nullable (backward compatible)
- All new models use ConfigDict(extra="forbid") and StrEnum conventions

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all models, repository methods, and signing utility are fully wired with no placeholder data.

## Self-Check: PASSED

All 7 created files verified on disk. All 4 commit hashes verified in git log.
