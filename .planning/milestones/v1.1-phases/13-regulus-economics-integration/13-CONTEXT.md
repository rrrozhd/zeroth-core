# Phase 13: Regulus Economics Integration - Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 13 integrates the Regulus economics backend into Zeroth so that every LLM provider call emits a cost event, token costs are attributed per node/run/tenant/deployment in audit records, per-tenant budget caps are enforced before execution begins, and cumulative cost totals are queryable via REST endpoints.

</domain>

<decisions>
## Implementation Decisions

### Regulus SDK Integration
- **D-01:** Use the `regulus-sdk` Python package as a typed dependency — provides typed client for event emission, budget queries, and configuration
- **D-02:** Regulus backend URL and API key configured via `ZerothSettings` (pydantic-settings, YAML + env vars) — follows Phase 11 config pattern
- **D-03:** Regulus client instantiated as a singleton dependency (like existing repository pattern) — injected where needed, not imported globally

### InstrumentedProviderAdapter Pattern
- **D-04:** `InstrumentedProviderAdapter` follows the decorator pattern — wraps any `ProviderAdapter` (including `LiteLLMProviderAdapter` and `GovernedLLMProviderAdapter`) and emits a Regulus `ExecutionEvent` after each `ainvoke()` call
- **D-05:** The orchestrator/runner does NOT change — adapter stacking handles instrumentation transparently. Stack order: `InstrumentedProviderAdapter(GovernedLLMProviderAdapter(LiteLLMProviderAdapter(...)))` or `InstrumentedProviderAdapter(LiteLLMProviderAdapter(...))` for ungoverned nodes
- **D-06:** ExecutionEvent includes: model_name, input_tokens, output_tokens, total_tokens, estimated_cost, node_id, run_id, tenant_id, deployment_ref, timestamp

### Cost Attribution in Audit Records
- **D-07:** New cost attribution fields added to `NodeAuditRecord`: `cost_usd` (Decimal or float), `cost_event_id` (Regulus event reference)
- **D-08:** `InstrumentedProviderAdapter` enriches `ProviderResponse` with cost data from the Regulus event response — runner copies cost fields to audit record alongside existing `token_usage` flow
- **D-09:** Cost attribution dimensions: node_id, run_id, tenant_id, deployment_ref — all already available in the execution context

### Budget Enforcement
- **D-10:** Pre-execution policy guard in `AgentRunner` — checks tenant budget cap against Regulus cumulative spend BEFORE calling the provider adapter
- **D-11:** Budget check is a fast Regulus API call (cached with short TTL) — not a database query
- **D-12:** Over-budget results in a policy rejection (reuses existing approval/policy rejection patterns) — no partial execution, clean failure before any tokens are consumed
- **D-13:** Budget caps configured per-tenant in Regulus backend — Zeroth reads caps, doesn't manage them

### Cost REST Endpoints
- **D-14:** `GET /v1/tenants/{id}/cost` — returns cumulative spend figure for the tenant, consistent with audit records
- **D-15:** `GET /v1/deployments/{ref}/cost` — deployment-level cost view (secondary)
- **D-16:** Cost endpoints query Regulus backend (source of truth), not local audit records — avoids inconsistency
- **D-17:** Endpoints follow existing FastAPI router patterns in `src/zeroth/service/`

### Testing
- **D-18:** Unit tests mock Regulus SDK client — validate event emission, cost enrichment, budget enforcement logic without Regulus backend
- **D-19:** Integration tests with Regulus backend gated behind `@pytest.mark.live` — similar to Phase 12's live LLM tests

### Claude's Discretion
- Exact Regulus SDK version pinning (use latest stable, pin minimum)
- ExecutionEvent field mapping details (follow SDK conventions)
- Budget cache TTL value (sensible default, e.g., 30s)
- Cost estimation logic (use Regulus pricing data or provider-reported costs)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Provider & Audit Architecture
- `src/zeroth/agent_runtime/provider.py` — ProviderAdapter protocol, LiteLLMProviderAdapter, GovernedLLMProviderAdapter, ProviderResponse with token_usage
- `src/zeroth/agent_runtime/runner.py` — AgentRunner retry loop, token_usage → audit record copy (lines 160-162)
- `src/zeroth/audit/models.py` — TokenUsage model (line 85), NodeAuditRecord with token_usage field (line 127)
- `src/zeroth/audit/repository.py` — AuditRepository async database pattern

### Configuration & Service Patterns
- `src/zeroth/config/` — ZerothSettings pydantic-settings pattern (Phase 11)
- `src/zeroth/service/audit_api.py` — Existing audit REST API patterns (FastAPI router)
- `src/zeroth/service/auth.py` — Tenant context extraction pattern

### Policy & Approval Patterns
- `src/zeroth/approvals/service.py` — Existing policy rejection patterns for budget enforcement reference

### Prior Phase Context
- `.planning/phases/12-real-llm-providers-retry/12-CONTEXT.md` — Phase 12 decisions (adapter architecture, token flow)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProviderAdapter` protocol (`provider.py:57`) — InstrumentedProviderAdapter implements this same protocol
- `GovernedLLMProviderAdapter` (`provider.py:93`) — Existing decorator pattern to follow
- `TokenUsage` model (`audit/models.py:85`) — Already captures input/output/total tokens
- `NodeAuditRecord` (`audit/models.py:101`) — Already has token_usage field, needs cost fields added
- `AuditRepository` — Async database pattern for querying audit data

### Established Patterns
- Decorator/wrapper adapter stacking (GovernedLLMProviderAdapter wraps LiteLLMProviderAdapter)
- Pydantic-settings config with YAML + env vars (Phase 11 ZerothSettings)
- FastAPI router patterns in `src/zeroth/service/`
- Async repository pattern for database access

### Integration Points
- `AgentRunner` — Budget check inserted before provider adapter call
- `ProviderResponse` — Cost data fields added alongside token_usage
- `NodeAuditRecord` — Cost attribution fields added
- `src/zeroth/service/` — New cost router registered alongside existing audit/auth routers

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Follow Regulus SDK conventions for event emission and budget queries.

</specifics>

<deferred>
## Deferred Ideas

- **ECON-05:** LLM response caching (semantic and exact-match) — separate phase
- **ECON-06:** Model routing and cost optimization — separate phase
- **ECON-07:** Regulus A/B experiments for model comparison — separate phase
- Model fallback chains (LLM-05) — deferred from Phase 12

</deferred>

---

*Phase: 13-regulus-economics-integration*
*Context gathered: 2026-04-07 via auto mode*
