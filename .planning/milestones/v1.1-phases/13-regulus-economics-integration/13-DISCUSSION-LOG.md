# Phase 13: Regulus Economics Integration - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-07
**Phase:** 13-regulus-economics-integration
**Areas discussed:** Regulus SDK integration, InstrumentedProviderAdapter design, Cost attribution model, Budget enforcement strategy, Cost API endpoints
**Mode:** Auto (all decisions auto-selected from recommended defaults)

---

## Regulus SDK Integration

| Option | Description | Selected |
|--------|-------------|----------|
| regulus-sdk package | Typed Python SDK dependency with client for events, budgets, config | ✓ |
| Raw HTTP client | Custom HTTP calls to Regulus API | |
| Mock-only (no real SDK) | Stub Regulus interface, implement later | |

**User's choice:** [auto] regulus-sdk Python package as typed dependency
**Notes:** Matches existing pydantic-settings pattern. Typed client provides SDK-level guarantees.

---

## InstrumentedProviderAdapter Design

| Option | Description | Selected |
|--------|-------------|----------|
| Decorator pattern | Wraps any ProviderAdapter, emits events after ainvoke() | ✓ |
| Runner-level hooks | Emit events directly in AgentRunner after provider call | |
| Middleware layer | Separate middleware between runner and adapter | |

**User's choice:** [auto] Decorator pattern wrapping ProviderAdapter
**Notes:** Non-invasive, follows existing GovernedLLMProviderAdapter pattern. Orchestrator untouched.

---

## Cost Attribution Model

| Option | Description | Selected |
|--------|-------------|----------|
| Enrich ProviderResponse | Adapter adds cost data to response, runner copies to audit | ✓ |
| Direct audit write | InstrumentedAdapter writes cost to audit record directly | |
| Post-process batch | Batch cost attribution from Regulus events after execution | |

**User's choice:** [auto] Enrich ProviderResponse with cost data, runner copies to NodeAuditRecord
**Notes:** Extends existing token_usage flow naturally. Single data path.

---

## Budget Enforcement Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Pre-execution guard | Check budget in AgentRunner before calling adapter | ✓ |
| Adapter-level check | InstrumentedAdapter checks budget before delegating | |
| Post-execution debit | Allow execution, debit after, reject next if over budget | |

**User's choice:** [auto] Pre-execution policy guard in AgentRunner
**Notes:** Fail-fast — no wasted tokens on over-budget calls. Clean rejection before execution.

---

## Cost API Endpoints

| Option | Description | Selected |
|--------|-------------|----------|
| Tenant + deployment endpoints | /v1/tenants/{id}/cost + /v1/deployments/{ref}/cost | ✓ |
| Tenant-only endpoint | /v1/tenants/{id}/cost only | |
| Aggregate cost dashboard | Multiple cost views with filtering | |

**User's choice:** [auto] Tenant and deployment cost endpoints querying Regulus backend
**Notes:** Matches existing FastAPI router patterns. Regulus is source of truth for cost data.

---

## Claude's Discretion

- Regulus SDK version pinning
- ExecutionEvent field mapping details
- Budget cache TTL value
- Cost estimation logic

## Deferred Ideas

- ECON-05: LLM response caching
- ECON-06: Model routing and cost optimization
- ECON-07: Regulus A/B experiments
- LLM-05: Model fallback chains
