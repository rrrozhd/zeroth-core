# External Integrations

**Analysis Date:** 2026-04-05

## APIs & External Services

**GovernAI (Core Governance Engine):**
- Local dependency at `file:///Users/dondoe/coding/governai`
- Provides: `GovernedLLM`, `Tool`, `PythonTool`, `GovernedStepSpec`, `RunState`, `RunStatus`
- Runtime stores: `RedisRunStore`, `RedisInterruptStore`, `RedisAuditEmitter`
- Tool call normalization: `NormalizedToolCall`, `extract_tool_calls`, `build_tool_message`
- Import locations: `src/zeroth/agent_runtime/provider.py`, `src/zeroth/storage/redis.py`, `src/zeroth/execution_units/`, `src/zeroth/runs/`, `src/zeroth/graph/`, `src/zeroth/contracts/`

**LLM Providers:**
- Accessed through GovernAI's `GovernedLLM` abstraction (`src/zeroth/agent_runtime/provider.py`)
- `GovernedLLMProviderAdapter` wraps `GovernedLLM` for production use
- `DeterministicProviderAdapter` available for testing without real LLM calls
- No direct OpenAI/Anthropic SDK imports; all LLM access is mediated through GovernAI

**JWT/OIDC Identity Providers:**
- JWKS-based token verification (`src/zeroth/service/auth.py`)
- Configurable issuer, audience, algorithms (default RS256)
- JWKS fetched from remote URL or provided inline via config
- Config via `ZEROTH_SERVICE_BEARER_JSON` env var

## API Surface (REST)

**Health & Diagnostics:**
- `GET /health` - Deployment health check (returns deployment ref, version, graph version ref)
- `GET /metrics` - Prometheus text exposition format metrics (requires `metrics:read` permission)

**Run Management (registered in `src/zeroth/service/run_api.py`):**
- Run submission and status endpoints
- Run lifecycle: PENDING -> RUNNING -> COMPLETED / FAILED / WAITING_INTERRUPT

**Admin Operations (registered in `src/zeroth/service/admin_api.py`):**
- `GET /admin/runs` - List runs by status (requires `run:admin`)
- `POST /admin/runs/{run_id}/cancel` - Force-fail a run
- `POST /admin/runs/{run_id}/replay` - Replay a dead-letter/failed run (resets to PENDING)
- `POST /admin/runs/{run_id}/interrupt` - Interrupt a running run (transitions to WAITING_INTERRUPT)

**Contracts (registered in `src/zeroth/service/contracts_api.py`):**
- Contract metadata endpoints

**Approvals (registered in `src/zeroth/service/approval_api.py`):**
- Human approval interaction endpoints

**Audit (registered in `src/zeroth/service/audit_api.py`):**
- Audit timeline, evidence, attestation endpoints

**Route Registration:**
- All routes registered in `src/zeroth/service/app.py` via `create_app()`
- Routes grouped by domain: runs, contracts, approvals, audit, admin/metrics

## Authentication & Identity

**Auth Mechanisms:**

1. **Static API Key** - `X-API-Key` header
   - Credentials loaded from `ZEROTH_SERVICE_API_KEYS_JSON` env var
   - Each key maps to: subject, roles, tenant_id, workspace_id
   - Model: `StaticApiKeyCredential` in `src/zeroth/service/auth.py`

2. **JWT Bearer Token** - `Authorization: Bearer <token>` header
   - OIDC-compatible with JWKS verification
   - Claims extracted: sub, roles, tenant_id, workspace_id
   - Verifier: `JWTBearerTokenVerifier` in `src/zeroth/service/auth.py`
   - Config via `ZEROTH_SERVICE_BEARER_JSON` env var

**Authorization (RBAC):**
- Three roles: `operator`, `reviewer`, `admin` (`src/zeroth/identity/models.py`)
- Permission enum in `src/zeroth/service/authorization.py`:
  - `deployment:read`, `run:create`, `run:read`
  - `approval:read`, `approval:resolve`
  - `audit:read`, `run:admin`, `metrics:read`
- Admin role has all permissions
- Operator: deployment read, run create/read, approval read
- Reviewer: deployment read, run read, approval read/resolve

**Identity Models (`src/zeroth/identity/models.py`):**
- `AuthenticatedPrincipal` - Shared identity shape across all services
- `ActorIdentity` - Actor tracking for audit trail
- `ServiceRole` enum - operator, reviewer, admin
- `AuthMethod` enum - API_KEY, BEARER
- `PrincipalScope` - Tenant and workspace scoping

**Authentication Middleware (`src/zeroth/service/app.py`):**
- Every HTTP request passes through auth middleware
- Failed auth records denial audit event via `record_service_denial()`
- Correlation ID propagated via `X-Correlation-ID` header (auto-generated if absent)

## Data Storage

**SQLite (Local Persistence):**
- Client: `SQLiteDatabase` wrapper in `src/zeroth/storage/sqlite.py`
- Features: WAL mode, foreign keys, versioned migrations, optional Fernet encryption
- Repositories using SQLite:
  - `src/zeroth/deployments/` - `SQLiteDeploymentRepository`
  - `src/zeroth/graph/` - `GraphRepository`
  - `src/zeroth/runs/` - `RunRepository`, `ThreadRepository`
  - `src/zeroth/approvals/` - `ApprovalRepository`
  - `src/zeroth/audit/` - `AuditRepository`
  - `src/zeroth/contracts/` - `ContractRegistry`

**Redis (Distributed Runtime State):**
- Client: `redis >=5.0.0` Python package
- Connection config: `RedisConfig` in `src/zeroth/storage/redis.py`
- Env vars: `ZEROTH_REDIS_HOST`, `ZEROTH_REDIS_PORT`, `ZEROTH_REDIS_MODE`, `ZEROTH_REDIS_PASSWORD`, `ZEROTH_REDIS_SSL`, `ZEROTH_REDIS_URL`, etc.
- Three GovernAI-backed store types:
  - `RedisRunStore` (prefix `zeroth:run`) - Run state tracking
  - `RedisInterruptStore` (prefix `zeroth:interrupt`) - Pause/approval handling
  - `RedisAuditEmitter` (prefix `zeroth:audit`) - Real-time audit events
- Factory: `build_governai_redis_runtime()` in `src/zeroth/storage/redis.py`
- Docker support: Automatic container detection via `docker inspect`

**File Storage:**
- Local filesystem only (SQLite files)

**Caching:**
- Redis serves as both runtime state store and implicit cache

## Monitoring & Observability

**Metrics:**
- Custom in-process `MetricsCollector` in `src/zeroth/observability/metrics.py`
- No external Prometheus client dependency; pure Python implementation
- Supports counters, gauges, histograms
- Renders Prometheus text exposition format via `render_prometheus_text()`
- Exposed at `GET /metrics` endpoint

**Queue Depth Gauge:**
- Background async task: `QueueDepthGauge` in `src/zeroth/observability/queue_gauge.py`
- Polls pending run count, reports `zeroth_queue_depth` gauge
- Configurable poll interval (default 10s)
- Started/stopped via FastAPI lifespan

**Correlation IDs:**
- Request-scoped correlation tracking in `src/zeroth/observability/correlation.py`
- Propagated via `X-Correlation-ID` HTTP header
- Functions: `get_correlation_id()`, `set_correlation_id()`, `new_correlation_id()`

**Logging:**
- Python standard `logging` module
- Pattern: `logger = logging.getLogger(__name__)` per module
- No structured logging library (no structlog, etc.)

**Error Tracking:**
- None (no Sentry, Datadog, or similar)

## Secrets Management

**Secret Provider Interface:**
- Protocol: `SecretProvider` in `src/zeroth/secrets/provider.py`
- Default: `EnvSecretProvider` reads from environment variables
- `SecretResolver` resolves `EnvironmentVariable` models into runtime env dicts
- `SecretRedactor` sanitizes secret values from output (`src/zeroth/secrets/redaction.py`)

**Field Encryption:**
- Fernet symmetric encryption for sensitive SQLite columns
- `EncryptedField` class in `src/zeroth/storage/sqlite.py`
- Key generation: `EncryptedField.generate_key()`

## Durable Dispatch & Guardrails

**Run Worker:**
- `RunWorker` in `src/zeroth/dispatch/worker.py` - Background poll loop for run execution
- Started via FastAPI lifespan in `src/zeroth/service/app.py`

**Lease Manager:**
- `LeaseManager` in `src/zeroth/dispatch/lease.py` - Run-level lease coordination
- Prevents duplicate execution; lease clearing on admin cancel

**Dead Letter:**
- `DeadLetterManager` in `src/zeroth/guardrails/dead_letter.py`
- Failed runs can be replayed via `POST /admin/runs/{run_id}/replay`

**Rate Limiting & Quotas:**
- `TokenBucketRateLimiter` in `src/zeroth/guardrails/rate_limit.py`
- `QuotaEnforcer` in `src/zeroth/guardrails/rate_limit.py`
- Config: `GuardrailConfig` in `src/zeroth/guardrails/config.py`

## CI/CD & Deployment

**Hosting:**
- Not configured (no Dockerfile, no deployment manifests)

**CI Pipeline:**
- Not detected (no `.github/workflows/` or CI config files)

## Webhooks & Callbacks

**Incoming:**
- None detected

**Outgoing:**
- None detected

## Environment Configuration

**Required env vars (for full service):**
- `ZEROTH_REDIS_*` - Redis connection (at minimum HOST/PORT or URL)
- `ZEROTH_SERVICE_API_KEYS_JSON` or `ZEROTH_SERVICE_BEARER_JSON` - Authentication config

**Secrets location:**
- Environment variables at runtime
- No `.env` file committed to repository

---

*Integration audit: 2026-04-05*
