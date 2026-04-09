# Phase 21: Health Probe Fix & Tech Debt - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Fix 4 specific technical issues: Regulus health check false-negative, Docker Compose missing env vars, missing agent_runtime re-exports, and stale Phase 14 verification status. Pure gap closure and tech debt — no new capabilities.

</domain>

<decisions>
## Implementation Decisions

### Regulus Health Fix
- **D-01:** Add `base_url` as a public read-only property on `RegulusClient` (in `src/zeroth/econ/client.py`). The health probe at `src/zeroth/service/health.py:155` already calls `getattr(regulus_client, "base_url", None)` — exposing the attribute makes this work without changing the health module. Store the `base_url` constructor arg as `self._base_url` and add a `@property` that returns it.
- **D-02:** No changes needed to `check_regulus()` or `register_health_routes()` — the existing logic is correct once `base_url` is accessible.

### Docker Compose Env Vars
- **D-03:** Add `ZEROTH_REGULUS__ENABLED: "true"` and `ZEROTH_REGULUS__BASE_URL: "http://regulus:8080/v1"` to the `zeroth` service `environment` block in `docker-compose.yml`. Follow the existing double-underscore env var naming pattern (e.g., `ZEROTH_DATABASE__BACKEND`).
- **D-04:** Point base URL to the `regulus` service hostname on port 8080 (matching `REGULUS_PORT` in the regulus service definition), with `/v1` path suffix (matching the `RegulusClient` default).

### Agent Runtime Re-exports
- **D-05:** Add exactly these 4 symbols to `src/zeroth/agent_runtime/__init__.py` imports and `__all__`:
  - `LiteLLMProviderAdapter` (from `provider.py`)
  - `MCPServerConfig` (from `mcp.py`)
  - `ModelParams` (from `models.py`)
  - `build_response_format` (from `response_format.py`)
- **D-06:** No other symbols need adding — scope limited to what's listed in success criteria.

### Phase 14 Verification Update
- **D-07:** Update `.planning/phases/14-memory-connectors-container-sandbox/14-VERIFICATION.md` to reflect that the ConnectorScope import error and MEM-06 requirement gap were resolved in Phase 18 cross-phase integration wiring. Change status from `gaps_found` to `passed` (or note partial resolution with specifics).

### Claude's Discretion
- Property implementation style (single `@property` vs storing as public attr) — either approach is fine
- Whether to add a test for the `base_url` property exposure — Claude can decide based on test patterns

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Regulus Health Fix
- `src/zeroth/econ/client.py` — RegulusClient class lacking base_url attribute (the bug)
- `src/zeroth/service/health.py` — Health probe that reads base_url via getattr (lines 153-155)

### Docker Compose
- `docker-compose.yml` — Service definitions, missing REGULUS env vars in zeroth service

### Agent Runtime Re-exports
- `src/zeroth/agent_runtime/__init__.py` — Current re-exports (missing 4 symbols)
- `src/zeroth/agent_runtime/provider.py` — Contains LiteLLMProviderAdapter
- `src/zeroth/agent_runtime/mcp.py` — Contains MCPServerConfig
- `src/zeroth/agent_runtime/models.py` — Contains ModelParams
- `src/zeroth/agent_runtime/response_format.py` — Contains build_response_format

### Stale Verification
- `.planning/phases/14-memory-connectors-container-sandbox/14-VERIFICATION.md` — Reports ConnectorScope and MEM-06 gaps
- `.planning/phases/18-cross-phase-integration-wiring/18-02-SUMMARY.md` — Phase 18 fixes that resolved Phase 14 gaps

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `RegulusClient` in `src/zeroth/econ/client.py` — thin wrapper, straightforward to add property
- Health probe infrastructure in `src/zeroth/service/health.py` — already structured for Regulus checks
- `agent_runtime/__init__.py` — well-organized with `__all__` list, easy to extend

### Established Patterns
- Env var naming: `ZEROTH_{SECTION}__{KEY}` double-underscore for pydantic-settings nesting
- Re-export pattern: import from submodule, add to `__all__` list alphabetically
- Health probes: `getattr(bootstrap, "...", None)` pattern for optional dependencies

### Integration Points
- `bootstrap.regulus_client` — where the RegulusClient instance lives at runtime
- `zeroth.config.settings` — Regulus settings likely already have `enabled` and `base_url` fields

</code_context>

<specifics>
## Specific Ideas

No specific requirements — this is a well-defined fix/cleanup phase with clear success criteria.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 21-health-probe-fix-tech-debt*
*Context gathered: 2026-04-09*
