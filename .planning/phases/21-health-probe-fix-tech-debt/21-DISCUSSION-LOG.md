# Phase 21: Health Probe Fix & Tech Debt - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-09
**Phase:** 21-health-probe-fix-tech-debt
**Areas discussed:** Regulus health fix approach, Docker Compose env strategy, Re-export scope, Verification update approach
**Mode:** --auto (all decisions auto-selected as recommended defaults)

---

## Regulus Health Fix Approach

| Option | Description | Selected |
|--------|-------------|----------|
| Add base_url as public property | Store constructor arg, expose via @property — minimal change | ✓ |
| Change health probe to read from settings | Health probe gets URL from config instead of client | |
| Store as public attribute directly | self.base_url = base_url in __init__ | |

**User's choice:** [auto] Add base_url as public property (recommended default)
**Notes:** Matches existing `getattr(regulus_client, "base_url", None)` pattern in health.py — no health module changes needed.

---

## Docker Compose Env Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Add ZEROTH_REGULUS__ENABLED and ZEROTH_REGULUS__BASE_URL | Follow existing double-underscore pattern | ✓ |
| Use .env file reference | Externalize to .env file for all Regulus config | |

**User's choice:** [auto] Add env vars directly to zeroth service (recommended default)
**Notes:** Matches existing env var pattern. Base URL points to regulus:8080/v1.

---

## Re-export Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Only the 4 listed symbols | LiteLLMProviderAdapter, MCPServerConfig, ModelParams, build_response_format | ✓ |
| Audit and add all missing symbols | Comprehensive re-export of everything in submodules | |

**User's choice:** [auto] Only the 4 listed symbols (recommended default)
**Notes:** Scope-limited to success criteria. Future phases can add more if needed.

---

## Verification Update Approach

| Option | Description | Selected |
|--------|-------------|----------|
| Update status to reflect Phase 18 fixes | Mark resolved gaps, update status | ✓ |
| Re-run full verification | Execute verification suite again from scratch | |

**User's choice:** [auto] Update status to reflect fixes (recommended default)
**Notes:** Phase 14 gaps (ConnectorScope import, MEM-06 requirement) were resolved in Phase 18.

---

## Claude's Discretion

- Property implementation style (@property vs public attribute)
- Whether to add a test for base_url property

## Deferred Ideas

None — phase is well-scoped technical fixes only.
