# Phase 15: Webhooks & Approval SLA - Research

**Researched:** 2026-04-07
**Domain:** Durable webhook delivery, approval SLA timeout enforcement
**Confidence:** HIGH

## Summary

Phase 15 adds two operational capabilities to Zeroth: (1) a durable webhook notification system that pushes HTTP POST callbacks to subscribers on run lifecycle events (completion, failure, approval requests) with retry and dead-letter guarantees, and (2) approval SLA timeout enforcement that escalates unactioned approvals rather than letting them hang indefinitely.

The codebase already has strong patterns for both capabilities. The `RunWorker` poll-loop pattern (dispatch/worker.py) provides a proven template for the webhook delivery worker and the SLA checker background task. The `DeadLetterManager` (guardrails/dead_letter.py) provides a failure-counting escalation pattern directly applicable to webhook dead-lettering. The `ApprovalService` (approvals/service.py) has a complete approval lifecycle that SLA timeout handling can extend. httpx (>=0.27, installed 0.28.1) is already a project dependency, making it the natural choice for outbound webhook HTTP requests.

**Primary recommendation:** Build webhook delivery and approval SLA as two independent subsystems following established patterns: new Pydantic models + async repositories + background poll-loop workers + Alembic migration for new tables. Wire into bootstrap and lifespan following Phase 9/13/14 patterns exactly.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Webhook subscriptions stored in Postgres via a new async `WebhookRepository` following the established async Database protocol pattern (Phase 11 D-07)
- **D-02:** A dedicated background delivery task (modeled after the existing `RunWorker` poll-loop pattern in `dispatch/worker.py`) polls for pending webhook deliveries and sends HTTP POST requests
- **D-03:** Webhook registration scoped per deployment -- subscribers register a URL for a specific deployment_ref and event types they want to receive
- **D-04:** Webhook payloads signed with HMAC-SHA256 using a per-subscription secret, delivered in a `X-Zeroth-Signature` header for subscriber verification
- **D-05:** Failed webhook deliveries retry with exponential backoff -- base delay 1s, max delay 5min, with jitter. Configurable max retry count (default 5)
- **D-06:** After max retries exhausted, the delivery is written to a dead-letter table (new Postgres table) rather than silently dropped
- **D-07:** Dead-letter entries are queryable via the admin API for operational visibility and manual replay
- **D-08:** Non-2xx HTTP responses and network timeouts are treated as delivery failures; only 2xx counts as successful delivery
- **D-09:** SLA timeout is a configurable duration on `HumanApprovalNode` (e.g., `sla_timeout: timedelta`) -- no SLA means no timeout (backward compatible)
- **D-10:** A periodic background task (approval SLA checker) polls for pending approvals past their SLA deadline -- similar poll-loop pattern to RunWorker
- **D-11:** Escalation action is configurable per approval node with three options: escalate to a configured delegate identity, auto-reject the approval, or raise an alert event (which triggers a webhook if subscribed)
- **D-12:** When an approval is escalated to a delegate, a new approval record is created for the delegate with the original context preserved
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

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| OPS-01 | Durable webhook notifications for run completion, approval needed, and failure events | Webhook subsystem: models, repository, delivery worker, retry/dead-letter, event emission hooks in orchestrator, admin API for subscriptions |
| OPS-02 | Approval SLA timeouts with escalation and delegation policies | SLA fields on HumanApprovalNodeData, SLA deadline tracking on ApprovalRecord, background SLA checker task, escalation logic (delegate/auto-reject/alert) |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| httpx | 0.28.1 | Outbound HTTP POST for webhook delivery | Already a project dependency; async-native; timeout/retry control built-in |
| pydantic | 2.10+ | Webhook/SLA data models | Project standard for all models |
| hmac + hashlib (stdlib) | 3.12 | HMAC-SHA256 payload signing | No external dependency needed; standard cryptographic signing |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| alembic | 1.18+ | Migration for new webhook/dead-letter/SLA tables | Already used for schema management |
| secrets (stdlib) | 3.12 | Generate per-subscription webhook secrets | `secrets.token_urlsafe(32)` for subscription secret generation |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| httpx | aiohttp | aiohttp is not a project dependency; httpx already in pyproject.toml and simpler API |
| Poll-loop delivery | Celery/ARQ task queue | Overkill; poll-loop is proven pattern in this codebase; ARQ planned for Phase 16 |

**Installation:** No new dependencies required. httpx, pydantic, alembic all already in pyproject.toml.

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/
├── webhooks/                    # New webhook subsystem
│   ├── __init__.py
│   ├── models.py                # WebhookSubscription, WebhookDelivery, WebhookDeadLetter, WebhookEvent enums, payload schemas
│   ├── repository.py            # WebhookRepository (subscriptions + deliveries + dead-letter)
│   ├── service.py               # WebhookService (create subscription, enqueue delivery, replay dead-letter)
│   ├── delivery.py              # WebhookDeliveryWorker (poll-loop background task)
│   └── signing.py               # HMAC-SHA256 signing utility
├── approvals/
│   ├── models.py                # Add SLA fields to ApprovalRecord, add ESCALATED status, add EscalationAction enum
│   ├── service.py               # Extend with SLA-aware creation, escalation methods
│   └── sla_checker.py           # NEW: ApprovalSLAChecker background task (poll-loop)
├── graph/
│   └── models.py                # Add sla_timeout and escalation_config to HumanApprovalNodeData
├── config/
│   └── settings.py              # Add WebhookSettings and ApprovalSLASettings sub-models
├── service/
│   ├── bootstrap.py             # Wire WebhookService, WebhookDeliveryWorker, ApprovalSLAChecker
│   ├── app.py                   # Start/stop delivery worker and SLA checker in lifespan
│   ├── webhook_api.py           # NEW: Webhook subscription CRUD + dead-letter query endpoints
│   └── admin_api.py             # Extend with dead-letter webhook replay endpoint
└── migrations/
    └── versions/
        └── 003_add_webhooks_and_sla.py  # New tables + approval column additions
```

### Pattern 1: Webhook Delivery Worker (Poll-Loop)
**What:** A background asyncio task that polls for pending webhook deliveries and sends HTTP POST requests, modeled after RunWorker.
**When to use:** For all outbound webhook delivery.
**Example:**
```python
# Modeled after dispatch/worker.py RunWorker.poll_loop
@dataclass
class WebhookDeliveryWorker:
    repository: WebhookRepository
    http_client: httpx.AsyncClient
    poll_interval: float = 2.0
    max_concurrency: int = 16

    async def poll_loop(self) -> None:
        while True:
            try:
                delivery = await self.repository.claim_pending_delivery()
                if delivery is not None:
                    asyncio.create_task(self._deliver(delivery))
                else:
                    await asyncio.sleep(self.poll_interval)
            except asyncio.CancelledError:
                raise
            except Exception:
                logger.exception("webhook delivery poll error")
                await asyncio.sleep(self.poll_interval)
```

### Pattern 2: HMAC-SHA256 Payload Signing
**What:** Sign webhook payloads so subscribers can verify authenticity.
**When to use:** Every webhook delivery.
**Example:**
```python
import hashlib
import hmac

def sign_payload(payload_bytes: bytes, secret: str) -> str:
    """Compute HMAC-SHA256 signature for webhook verification."""
    return hmac.new(
        secret.encode("utf-8"),
        payload_bytes,
        hashlib.sha256,
    ).hexdigest()

# Set header: X-Zeroth-Signature: sha256={signature}
```

### Pattern 3: Exponential Backoff with Jitter
**What:** Retry failed webhook deliveries with increasing delays and randomization.
**When to use:** On every delivery failure before dead-lettering.
**Example:**
```python
import random

def next_retry_delay(attempt: int, base: float = 1.0, max_delay: float = 300.0) -> float:
    """Exponential backoff with full jitter. attempt is 0-indexed."""
    delay = min(base * (2 ** attempt), max_delay)
    return random.uniform(0, delay)  # Full jitter per AWS best practices
```

### Pattern 4: Approval SLA Checker (Poll-Loop)
**What:** Background task polling for pending approvals past their SLA deadline.
**When to use:** Runs alongside the main worker in the lifespan.
**Example:**
```python
@dataclass
class ApprovalSLAChecker:
    approval_service: ApprovalService
    webhook_service: WebhookService | None
    poll_interval: float = 10.0

    async def poll_loop(self) -> None:
        while True:
            try:
                overdue = await self.approval_service.list_overdue()
                for record in overdue:
                    await self._escalate(record)
            except asyncio.CancelledError:
                raise
            except Exception:
                logger.exception("SLA checker poll error")
            await asyncio.sleep(self.poll_interval)
```

### Pattern 5: Event Emission Hook
**What:** Emit webhook events from orchestrator/service layer at state transitions.
**When to use:** After run completion/failure, approval creation/resolution/escalation.
**Example:**
```python
# In orchestrator after run completes:
if webhook_service is not None:
    await webhook_service.emit_event(
        event_type="run.completed",
        deployment_ref=run.deployment_ref,
        tenant_id=run.tenant_id,
        data={"run_id": run.run_id, "status": "completed", ...},
    )
```

### Anti-Patterns to Avoid
- **Inline delivery in request path:** Never send webhooks synchronously during API request handling. Always enqueue for background delivery.
- **Unbounded retries:** Always cap retry attempts (default 5) and dead-letter after exhaustion.
- **Shared HTTP client without limits:** Configure `httpx.AsyncClient` with `limits=httpx.Limits(max_connections=100, max_keepalive_connections=20)` and `timeout=httpx.Timeout(10.0)`.
- **Blocking SLA check:** SLA checker should not hold database transactions open while performing escalation actions.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP client | Custom urllib/requests wrapper | httpx.AsyncClient | Connection pooling, timeouts, async-native, already in project |
| HMAC signing | Custom crypto | stdlib hmac + hashlib | Standard, auditable, no dependency |
| Retry delay calculation | Custom timer logic | Simple math function (2^n with jitter) | AWS-documented pattern, ~5 lines of code |
| JSON serialization | Custom serializer | Pydantic model_dump(mode="json") + to_json_value | Project-standard pattern in storage/json.py |
| Background task lifecycle | Custom thread pool | asyncio.create_task in FastAPI lifespan | Established pattern in service/app.py |

**Key insight:** The codebase already has all the patterns needed. The webhook system is essentially "RunWorker for HTTP POST instead of orchestrator.drive" and "DeadLetterManager for webhook deliveries instead of runs."

## Common Pitfalls

### Pitfall 1: Webhook Delivery Timeout vs Poll Interval
**What goes wrong:** If the HTTP timeout for webhook delivery exceeds the poll interval, the worker can accumulate unbounded concurrent requests.
**Why it happens:** Default httpx timeout is 5s, but slow subscribers can take 10-30s.
**How to avoid:** Use a semaphore (like RunWorker) to bound concurrency. Set explicit httpx timeout (10s max). Track active deliveries.
**Warning signs:** Memory growth, connection pool exhaustion.

### Pitfall 2: HMAC Timing Attack
**What goes wrong:** Using `==` to compare HMAC signatures allows timing-based attacks.
**Why it happens:** String comparison short-circuits on first mismatch.
**How to avoid:** Use `hmac.compare_digest()` for signature verification. (Relevant if we document subscriber-side verification guidance, less relevant server-side since we generate the signature.)
**Warning signs:** N/A -- preventive measure.

### Pitfall 3: SLA Checker Double-Escalation
**What goes wrong:** The SLA checker escalates the same approval multiple times if the poll loop fires before the first escalation completes.
**Why it happens:** Race condition between checking "overdue" and marking "escalated."
**How to avoid:** Add an `ESCALATED` status to ApprovalStatus. The escalation operation atomically transitions PENDING -> ESCALATED. The query for overdue approvals only returns PENDING status.
**Warning signs:** Duplicate escalation records, duplicate webhook events.

### Pitfall 4: Stale Delivery Claims
**What goes wrong:** Two workers claim the same pending delivery if using a simple SELECT + UPDATE pattern.
**Why it happens:** No atomic claim mechanism.
**How to avoid:** Use an atomic UPDATE ... WHERE status = 'pending' AND next_attempt_at <= now() LIMIT 1 RETURNING pattern, or a single-worker model since webhook delivery is append-only.
**Warning signs:** Duplicate webhook deliveries to subscribers.

### Pitfall 5: Migration Column Addition on Existing Tables
**What goes wrong:** Adding NOT NULL columns to existing tables (approvals) without defaults breaks existing rows.
**Why it happens:** Alembic ALTER TABLE with NOT NULL constraint on populated table.
**How to avoid:** Add new columns as nullable with defaults, or use server_default. SLA fields should be nullable (NULL = no SLA configured, backward compatible per D-09).
**Warning signs:** Migration fails on non-empty database.

### Pitfall 6: Circular Import Between Webhooks and Approvals
**What goes wrong:** WebhookService imported in ApprovalService for event emission, and vice versa.
**Why it happens:** SLA escalation needs to emit webhook events, and approval events need to trigger webhooks.
**How to avoid:** Use the established pattern: pass WebhookService as an optional dependency to ApprovalSLAChecker (like dead_letter_manager is optional on RunWorker). The SLA checker is the integration point, not the service itself.
**Warning signs:** ImportError on startup.

## Code Examples

### Webhook Subscription Model
```python
# Following project patterns: ConfigDict(extra="forbid"), StrEnum, _utc_now, _new_id
from enum import StrEnum
from pydantic import BaseModel, ConfigDict, Field

class WebhookEventType(StrEnum):
    RUN_COMPLETED = "run.completed"
    RUN_FAILED = "run.failed"
    APPROVAL_REQUESTED = "approval.requested"
    APPROVAL_RESOLVED = "approval.resolved"
    APPROVAL_ESCALATED = "approval.escalated"

class WebhookSubscription(BaseModel):
    model_config = ConfigDict(extra="forbid")
    subscription_id: str = Field(default_factory=_new_id)
    deployment_ref: str
    tenant_id: str = "default"
    target_url: str
    secret: str = Field(default_factory=lambda: secrets.token_urlsafe(32))
    event_types: list[WebhookEventType]
    active: bool = True
    created_at: datetime = Field(default_factory=_utc_now)
    updated_at: datetime = Field(default_factory=_utc_now)
```

### Webhook Delivery Model
```python
class DeliveryStatus(StrEnum):
    PENDING = "pending"
    DELIVERED = "delivered"
    FAILED = "failed"
    DEAD_LETTER = "dead_letter"

class WebhookDelivery(BaseModel):
    model_config = ConfigDict(extra="forbid")
    delivery_id: str = Field(default_factory=_new_id)
    subscription_id: str
    event_type: WebhookEventType
    event_id: str = Field(default_factory=_new_id)
    payload_json: str  # Pre-serialized JSON for signing consistency
    status: DeliveryStatus = DeliveryStatus.PENDING
    attempt_count: int = 0
    max_attempts: int = 5
    next_attempt_at: datetime = Field(default_factory=_utc_now)
    last_error: str | None = None
    last_status_code: int | None = None
    created_at: datetime = Field(default_factory=_utc_now)
    updated_at: datetime = Field(default_factory=_utc_now)
```

### SLA Fields on HumanApprovalNodeData
```python
# Addition to existing HumanApprovalNodeData in graph/models.py
class EscalationAction(StrEnum):
    DELEGATE = "delegate"
    AUTO_REJECT = "auto_reject"
    ALERT = "alert"

class HumanApprovalNodeData(BaseModel):
    # ... existing fields ...
    sla_timeout_seconds: int | None = None  # None = no SLA (backward compatible)
    escalation_action: EscalationAction | None = None
    delegate_identity: dict[str, Any] | None = None  # ActorIdentity-compatible dict
```

### SLA Fields on ApprovalRecord
```python
# Additional fields for ApprovalRecord in approvals/models.py
class ApprovalStatus(StrEnum):
    PENDING = "pending"
    RESOLVED = "resolved"
    ESCALATED = "escalated"  # NEW: prevents double-escalation

# New fields on ApprovalRecord:
#   sla_deadline: datetime | None = None  # Computed at creation: created_at + sla_timeout
#   escalation_action: str | None = None  # From node config
#   escalated_from_id: str | None = None  # Links delegate approval to original
```

### Alembic Migration Pattern (003)
```python
revision = "003"
down_revision = "002"

def upgrade() -> None:
    # Webhook subscriptions table
    op.execute("""
        CREATE TABLE IF NOT EXISTS webhook_subscriptions (
            subscription_id TEXT PRIMARY KEY,
            deployment_ref TEXT NOT NULL,
            tenant_id TEXT NOT NULL DEFAULT 'default',
            target_url TEXT NOT NULL,
            secret TEXT NOT NULL,
            event_types TEXT NOT NULL,  -- JSON array
            active INTEGER NOT NULL DEFAULT 1,
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL
        )
    """)
    op.execute("""
        CREATE INDEX IF NOT EXISTS idx_webhook_subs_deployment
        ON webhook_subscriptions(deployment_ref, active)
    """)

    # Webhook deliveries table
    op.execute("""
        CREATE TABLE IF NOT EXISTS webhook_deliveries (
            delivery_id TEXT PRIMARY KEY,
            subscription_id TEXT NOT NULL,
            event_type TEXT NOT NULL,
            event_id TEXT NOT NULL,
            payload_json TEXT NOT NULL,
            status TEXT NOT NULL DEFAULT 'pending',
            attempt_count INTEGER NOT NULL DEFAULT 0,
            max_attempts INTEGER NOT NULL DEFAULT 5,
            next_attempt_at TEXT NOT NULL,
            last_error TEXT,
            last_status_code INTEGER,
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL,
            FOREIGN KEY (subscription_id) REFERENCES webhook_subscriptions(subscription_id)
        )
    """)
    op.execute("""
        CREATE INDEX IF NOT EXISTS idx_webhook_del_pending
        ON webhook_deliveries(status, next_attempt_at)
    """)

    # Dead-letter table for exhausted deliveries
    op.execute("""
        CREATE TABLE IF NOT EXISTS webhook_dead_letters (
            dead_letter_id TEXT PRIMARY KEY,
            delivery_id TEXT NOT NULL,
            subscription_id TEXT NOT NULL,
            event_type TEXT NOT NULL,
            event_id TEXT NOT NULL,
            payload_json TEXT NOT NULL,
            attempt_count INTEGER NOT NULL,
            last_error TEXT,
            last_status_code INTEGER,
            created_at TEXT NOT NULL,
            dead_lettered_at TEXT NOT NULL
        )
    """)
    op.execute("""
        CREATE INDEX IF NOT EXISTS idx_webhook_dl_subscription
        ON webhook_dead_letters(subscription_id, dead_lettered_at DESC)
    """)

    # Add SLA columns to approvals table
    op.add_column("approvals", sa.Column("sla_deadline", sa.Text(), nullable=True))
    op.add_column("approvals", sa.Column("escalation_action", sa.Text(), nullable=True))
    op.add_column("approvals", sa.Column("escalated_from_id", sa.Text(), nullable=True))
```

### WebhookSettings Configuration
```python
class WebhookSettings(BaseModel):
    """Webhook delivery configuration."""
    enabled: bool = True
    delivery_poll_interval: float = 2.0
    delivery_timeout: float = 10.0
    max_delivery_concurrency: int = 16
    default_max_retries: int = 5
    retry_base_delay: float = 1.0
    retry_max_delay: float = 300.0

class ApprovalSLASettings(BaseModel):
    """Approval SLA checker configuration."""
    enabled: bool = True
    checker_poll_interval: float = 10.0
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Synchronous webhook delivery | Async with background worker | Standard practice | Decouples delivery from request path |
| Simple retry (fixed delay) | Exponential backoff with jitter | AWS best practice, widely adopted | Prevents thundering herd on subscriber recovery |
| Silent failure on webhook exhaustion | Dead-letter store + admin API | Standard in event-driven systems | Operational visibility, manual replay |
| No approval timeout | SLA with escalation policy | N/A (new feature) | Prevents approvals from blocking indefinitely |

## Open Questions

1. **Delivery ordering guarantees**
   - What we know: Deliveries are enqueued per-event, claimed in next_attempt_at order
   - What's unclear: Whether subscribers need strict per-deployment ordering
   - Recommendation: Best-effort ordering by created_at. Document that concurrent retries may deliver out of order. Not worth implementing strict ordering for v1.1.

2. **Webhook subscription management permissions**
   - What we know: Admin API uses `Permission.RUN_ADMIN` for run management
   - What's unclear: Whether webhook subscriptions need a separate permission
   - Recommendation: Add `Permission.WEBHOOK_ADMIN` to the existing StrEnum. Webhook CRUD requires this permission. Keep it simple.

3. **SLA checker and webhook delivery: one task or two?**
   - What we know: RunWorker and QueueDepthGauge already run as separate background tasks
   - What's unclear: Whether to merge webhook delivery and SLA checking
   - Recommendation: **Separate tasks.** Different poll intervals (2s for delivery, 10s for SLA). Different failure modes. Follows established pattern of one-task-per-concern.

## Project Constraints (from CLAUDE.md)

- **Build/test:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** `src/zeroth/` for main package, `tests/` for pytest tests
- **Progress logging:** Every implementation session MUST use progress-logger skill
- **Testing:** pytest with asyncio_mode="auto", live tests gated behind `@pytest.mark.live`
- **Code conventions:** Pydantic models with `ConfigDict(extra="forbid")`, `StrEnum` for enums, `_utc_now()` and `_new_id()` helpers, async repository pattern
- **Storage pattern:** AsyncDatabase protocol with `transaction()` context manager, `?`-placeholder SQL (SQLite-compatible), `to_json_value()`/`load_typed_value()` for JSON columns

## Sources

### Primary (HIGH confidence)
- Codebase analysis: `src/zeroth/dispatch/worker.py` -- RunWorker poll-loop pattern (template for delivery worker and SLA checker)
- Codebase analysis: `src/zeroth/guardrails/dead_letter.py` -- DeadLetterManager pattern (template for webhook dead-lettering)
- Codebase analysis: `src/zeroth/approvals/models.py`, `service.py`, `repository.py` -- Full approval lifecycle (extension point for SLA)
- Codebase analysis: `src/zeroth/service/bootstrap.py`, `app.py` -- Wiring and lifespan patterns
- Codebase analysis: `src/zeroth/storage/database.py` -- AsyncDatabase/AsyncConnection protocol
- Codebase analysis: `src/zeroth/config/settings.py` -- Settings sub-model pattern
- Codebase analysis: `src/zeroth/migrations/versions/` -- Alembic migration pattern (001 initial, 002 incremental)
- pyproject.toml: httpx>=0.27 already a dependency; installed version 0.28.1

### Secondary (MEDIUM confidence)
- httpx documentation: AsyncClient with Limits and Timeout configuration
- Python stdlib hmac: HMAC-SHA256 signing with compare_digest for timing-safe comparison

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in project, no new dependencies
- Architecture: HIGH -- all patterns directly derived from existing codebase (RunWorker, DeadLetterManager, ApprovalService)
- Pitfalls: HIGH -- derived from concrete codebase patterns and standard webhook delivery concerns

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable domain, no fast-moving dependencies)
