# Phase 15: Webhooks & Approval SLA - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-07
**Phase:** 15-webhooks-approval-sla
**Areas discussed:** Webhook delivery, Retry & dead-letter, Approval SLA, Event taxonomy
**Mode:** Auto (--auto flag, all recommended defaults selected)

---

## Webhook Delivery Model

| Option | Description | Selected |
|--------|-------------|----------|
| Background delivery task | Dedicated async background task polls for pending deliveries, sends HTTP POST (follows RunWorker pattern) | [auto] |
| Inline delivery | Send webhooks synchronously during run completion (simpler but blocks orchestrator) | |
| Queue-based delivery | Push to Redis/ARQ queue for async processing | |

**User's choice:** [auto] Background delivery task (recommended default)
**Notes:** Matches existing RunWorker poll-loop pattern. Avoids blocking orchestrator. Queue-based deferred to Phase 16 ARQ integration.

---

## Retry & Dead-Letter Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Exponential backoff with dead-letter store | Retry with backoff (1s base, 5min max, jitter), dead-letter after max retries (default 5) | [auto] |
| Fixed interval retry | Retry at fixed intervals, simpler but less polite to receivers | |
| Fire-and-forget | No retries, log failures only | |

**User's choice:** [auto] Exponential backoff with dead-letter store (recommended default)
**Notes:** Aligns with success criteria requiring retries and dead-letter rather than silent drops. Extends existing DeadLetterManager pattern.

---

## Approval SLA Mechanism

| Option | Description | Selected |
|--------|-------------|----------|
| Background poller with configurable SLA | Periodic task checks pending approvals past deadline, escalation configurable per node | [auto] |
| Database trigger / scheduled job | DB-level timeout detection | |
| Event-driven with TTL | Set TTL on approval records, process expirations | |

**User's choice:** [auto] Background poller with configurable SLA (recommended default)
**Notes:** Consistent with existing poll-loop patterns. SLA timeout on HumanApprovalNode keeps config close to the graph definition. Three escalation actions: delegate, auto-reject, alert.

---

## Event Taxonomy

| Option | Description | Selected |
|--------|-------------|----------|
| Core lifecycle events | run.completed, run.failed, approval.requested, approval.resolved, approval.escalated | [auto] |
| Minimal events | run.completed, run.failed only | |
| Comprehensive events | All lifecycle + node-level events (node.started, node.completed) | |

**User's choice:** [auto] Core lifecycle events (recommended default)
**Notes:** Covers all three success criteria. HMAC-SHA256 signature for verification. Standard payload schema with event_type, event_id, timestamp, deployment_ref, tenant_id, and event-specific data.

---

## Claude's Discretion

- Table schemas, HTTP client choice, poll intervals, migration structure, admin API endpoint design

## Deferred Ideas

None -- discussion stayed within phase scope.
