---
phase: 15-webhooks-approval-sla
plan: 02
subsystem: webhooks
tags: [webhooks, delivery, service, api, rest, hmac, retry, dead-letter]
dependency_graph:
  requires: [webhook-models, webhook-repository, webhook-signing]
  provides: [webhook-service, webhook-delivery-worker, webhook-api, webhook-admin-permission]
  affects: [service-app, service-authorization, service-bootstrap]
tech_stack:
  added: [httpx]
  patterns: [poll-loop-worker, exponential-backoff-jitter, hmac-signing, semaphore-concurrency]
key_files:
  created:
    - src/zeroth/webhooks/service.py
    - src/zeroth/webhooks/delivery.py
    - src/zeroth/service/webhook_api.py
    - tests/test_webhook_service.py
    - tests/test_webhook_delivery.py
    - tests/test_webhook_api.py
  modified:
    - src/zeroth/service/app.py
    - src/zeroth/service/authorization.py
    - src/zeroth/webhooks/__init__.py
decisions:
  - WebhookDeliveryWorker uses semaphore-based bounded concurrency matching RunWorker pattern
  - WEBHOOK_ADMIN permission auto-included in ADMIN role via set(Permission)
  - Dead-letter replay creates fresh delivery with reset attempt_count
metrics:
  duration: 413s
  completed: "2026-04-07T12:58:49Z"
  tasks: 2
  files: 9
---

# Phase 15 Plan 02: Webhook Delivery Pipeline Summary

WebhookService emits events by enqueuing deliveries per matching subscription; WebhookDeliveryWorker sends HTTP POST with HMAC-SHA256 X-Zeroth-Signature header, retries with jittered exponential backoff, and dead-letters after max attempts exhausted; REST API provides subscription CRUD and dead-letter management with WEBHOOK_ADMIN permission enforcement.

## Tasks Completed

### Task 1: WebhookService and WebhookDeliveryWorker (TDD)

**Commits:** `6e816a5` (RED), `62d64cb` (GREEN)

- `WebhookService`: `emit_event` finds active subscriptions matching deployment_ref + event_type, builds `WebhookEventPayload` with event_id/timestamp/data, enqueues a `WebhookDelivery` per subscription
- `WebhookService`: `create_subscription`, `list_subscriptions`, `deactivate_subscription`, `delete_subscription` delegate to repository
- `WebhookService`: `replay_dead_letter` re-enqueues dead-letter entry as new PENDING delivery
- `WebhookDeliveryWorker`: poll-loop pattern modeled on `RunWorker` with `asyncio.Semaphore` for bounded concurrency
- `WebhookDeliveryWorker._deliver`: sends HTTP POST with `Content-Type: application/json`, `X-Zeroth-Signature: sha256=...`, `X-Zeroth-Event`, `X-Zeroth-Delivery` headers
- `WebhookDeliveryWorker._handle_failure`: retries with `next_retry_delay` (jittered exponential backoff), dead-letters when `attempt_count + 1 >= max_attempts`
- `next_retry_delay`: returns `random.uniform(0, min(base * 2^attempt, max_delay))`
- 19 tests: emit_event matching/no-match/payload/inactive, subscription CRUD, replay, HTTP delivery success/500/timeout/dead-letter, poll-loop sleep, backoff bounds

### Task 2: Webhook REST API, Permission, lifespan wiring

**Commit:** `feea102`

- Added `WEBHOOK_ADMIN = "webhook:admin"` to `Permission` StrEnum (automatically in ADMIN role via `set(Permission)`)
- Created `src/zeroth/service/webhook_api.py` with 6 endpoints:
  - `POST /webhooks/subscriptions` (201) -- create subscription
  - `GET /webhooks/subscriptions` -- list with optional deployment_ref/tenant_id filters
  - `GET /webhooks/subscriptions/{subscription_id}` -- get single subscription
  - `DELETE /webhooks/subscriptions/{subscription_id}` (204) -- deactivate (soft delete)
  - `GET /webhooks/dead-letters` -- list dead-letter entries
  - `POST /webhooks/dead-letters/{dead_letter_id}/replay` (201) -- replay dead-letter
- All endpoints require `WEBHOOK_ADMIN` permission
- Updated `src/zeroth/service/app.py`: registered webhook routes, added delivery worker to lifespan (start/cancel), added httpx client cleanup
- Updated `src/zeroth/webhooks/__init__.py` to export `WebhookService` and `WebhookDeliveryWorker`
- 10 tests: create/list/get/deactivate subscriptions, list/replay dead-letters, 404 cases, permission enforcement

## Deviations from Plan

None - plan executed exactly as written.

## Verification

```
uv run pytest tests/test_webhook_service.py tests/test_webhook_delivery.py tests/test_webhook_api.py -x -v
29 passed

uv run ruff check src/zeroth/webhooks/ src/zeroth/service/webhook_api.py src/zeroth/service/app.py src/zeroth/service/authorization.py
All checks passed!
```

## Known Stubs

None. All service methods are fully wired to the repository layer. The delivery worker requires `bootstrap.delivery_worker` and `bootstrap.webhook_http_client` to be set in bootstrap (will be wired in a future plan or bootstrap update).

## Self-Check: PASSED
