# Phase 15: Webhooks & Approval SLA - Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 15 delivers two capabilities: (1) a durable webhook notification system that pushes HTTP POST callbacks to subscribers on run lifecycle events (completion, failure, approval requests), with retry and dead-letter guarantees; and (2) approval SLA timeout enforcement that escalates unactioned approvals rather than letting them hang indefinitely.

</domain>

<decisions>
## Implementation Decisions

### Webhook Delivery Model
- **D-01:** Webhook subscriptions stored in Postgres via a new async `WebhookRepository` following the established async Database protocol pattern (Phase 11 D-07)
- **D-02:** A dedicated background delivery task (modeled after the existing `RunWorker` poll-loop pattern in `dispatch/worker.py`) polls for pending webhook deliveries and sends HTTP POST requests
- **D-03:** Webhook registration scoped per deployment — subscribers register a URL for a specific deployment_ref and event types they want to receive
- **D-04:** Webhook payloads signed with HMAC-SHA256 using a per-subscription secret, delivered in a `X-Zeroth-Signature` header for subscriber verification

### Retry & Dead-Letter Strategy
- **D-05:** Failed webhook deliveries retry with exponential backoff — base delay 1s, max delay 5min, with jitter. Configurable max retry count (default 5)
- **D-06:** After max retries exhausted, the delivery is written to a dead-letter table (new Postgres table) rather than silently dropped
- **D-07:** Dead-letter entries are queryable via the admin API for operational visibility and manual replay
- **D-08:** Non-2xx HTTP responses and network timeouts are treated as delivery failures; only 2xx counts as successful delivery

### Approval SLA Mechanism
- **D-09:** SLA timeout is a configurable duration on `HumanApprovalNode` (e.g., `sla_timeout: timedelta`) — no SLA means no timeout (backward compatible)
- **D-10:** A periodic background task (approval SLA checker) polls for pending approvals past their SLA deadline — similar poll-loop pattern to RunWorker
- **D-11:** Escalation action is configurable per approval node with three options: escalate to a configured delegate identity, auto-reject the approval, or raise an alert event (which triggers a webhook if subscribed)
- **D-12:** When an approval is escalated to a delegate, a new approval record is created for the delegate with the original context preserved

### Event Taxonomy
- **D-13:** Core webhook event types: `run.completed`, `run.failed`, `approval.requested`, `approval.resolved`, `approval.escalated`
- **D-14:** Standard payload schema: `{ event_type, event_id, timestamp, deployment_ref, tenant_id, data: { ...event-specific fields } }`
- **D-15:** `run.completed` and `run.failed` payloads include run_id, graph_version_ref, duration, and final status. `approval.*` payloads include approval_id, run_id, node_id, and SLA metadata

### Claude's Discretion
- Exact Postgres table schemas for webhook subscriptions, deliveries, and dead-letter entries
- Connection pooling and HTTP client configuration for outbound webhook requests (httpx vs aiohttp)
- Poll interval tuning for both the webhook delivery worker and approval SLA checker
- Whether webhook delivery and SLA checking share a single background task or run as separate tasks
- Alembic migration structure for new tables
- Admin API endpoint design for webhook subscription management and dead-letter inspection

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Approval System
- `src/zeroth/approvals/service.py` -- ApprovalService with full lifecycle (create_pending, resolve). SLA timeout extends this service
- `src/zeroth/approvals/models.py` -- ApprovalRecord, ApprovalStatus, ApprovalDecision, HumanInteractionType. SLA fields will be added here
- `src/zeroth/approvals/repository.py` -- ApprovalRepository for persistence
- `src/zeroth/service/approval_api.py` -- Approval REST endpoints

### Existing Dispatch & Dead-Letter Patterns
- `src/zeroth/dispatch/worker.py` -- RunWorker poll-loop pattern. Webhook delivery worker should follow this structure
- `src/zeroth/dispatch/lease.py` -- LeaseManager for concurrency control
- `src/zeroth/guardrails/dead_letter.py` -- DeadLetterManager pattern for escalation after repeated failures

### Storage & Config Patterns
- `src/zeroth/storage/sqlite.py` -- Current async Database interface
- `src/zeroth/config/` -- Unified pydantic-settings configuration (Phase 11). New webhook/SLA settings go here
- `src/zeroth/migrations/versions/001_initial_schema.py` -- Alembic migration pattern

### Service Layer
- `src/zeroth/service/bootstrap.py` -- ServiceBootstrap composition root. New webhook/SLA services wired here
- `src/zeroth/service/app.py` -- FastAPI app factory, lifespan for background task startup
- `src/zeroth/service/admin_api.py` -- Admin endpoints pattern for dead-letter and webhook management

### Graph Models
- `src/zeroth/graph/models.py` -- HumanApprovalNode definition where SLA config fields will be added

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `RunWorker` (dispatch/worker.py): Poll-loop background task pattern with lease management — reuse for webhook delivery worker
- `DeadLetterManager` (guardrails/dead_letter.py): Escalation-after-failure pattern — extend for webhook dead-letter
- `ApprovalService` (approvals/service.py): Full approval lifecycle — extend with SLA timeout handling
- `ApprovalRecord` (approvals/models.py): Already has `urgency_metadata` dict field — can carry SLA-related data
- `ServiceBootstrap` (service/bootstrap.py): Composition root pattern — new services wire in here
- Async repository pattern from Phase 11 — all new repositories follow this

### Established Patterns
- Background tasks started/stopped in FastAPI lifespan (`service/app.py`)
- Pydantic models with `ConfigDict(extra="forbid")` for strict validation
- `StrEnum` for all enums with lowercase string values
- Repository pattern: `async def get()`, `async def put()`, `async def list_by_*()`
- `_utc_now()` and `_new_id()` private helper conventions

### Integration Points
- `service/app.py` lifespan: Start webhook delivery worker and SLA checker background tasks
- `service/bootstrap.py`: Wire WebhookService, WebhookRepository, SLA checker into bootstrap
- `orchestrator/runtime.py`: Emit webhook events after run completion/failure and approval creation
- `approvals/service.py`: Hook SLA timeout into approval creation flow (set deadline)
- `graph/models.py`: Add SLA config fields to HumanApprovalNode
- `config/`: Add WebhookSettings and ApprovalSLASettings sub-models

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches.
Auto-mode selected recommended defaults for all gray areas.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 15-webhooks-approval-sla*
*Context gathered: 2026-04-07*
