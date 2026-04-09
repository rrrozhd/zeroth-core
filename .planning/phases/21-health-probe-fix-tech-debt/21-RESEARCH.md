# Phase 21: Health Probe Fix & Tech Debt - Research

**Researched:** 2026-04-09
**Domain:** Bug fixes, Docker configuration, module re-exports, documentation cleanup
**Confidence:** HIGH

## Summary

Phase 21 addresses four discrete tech debt items: (1) a false-negative in the Regulus health probe caused by `RegulusClient` not exposing `base_url` as a readable attribute, (2) missing Regulus environment variables in `docker-compose.yml`, (3) four missing re-exports from `agent_runtime/__init__.py`, and (4) a stale Phase 14 verification document that still reports gaps resolved in Phase 18.

All four items are well-scoped, independent, and require no new dependencies. The existing code patterns are clear and the fixes are mechanical.

**Primary recommendation:** Execute all four fixes in a single plan with one task per fix. Each is independent and low-risk.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Add `base_url` as a public read-only property on `RegulusClient` (in `src/zeroth/econ/client.py`). Store the `base_url` constructor arg as `self._base_url` and add a `@property` that returns it.
- D-02: No changes needed to `check_regulus()` or `register_health_routes()` -- the existing logic is correct once `base_url` is accessible.
- D-03: Add `ZEROTH_REGULUS__ENABLED: "true"` and `ZEROTH_REGULUS__BASE_URL: "http://regulus:8080/v1"` to the `zeroth` service `environment` block in `docker-compose.yml`.
- D-04: Point base URL to the `regulus` service hostname on port 8080 with `/v1` path suffix.
- D-05: Add exactly 4 symbols to `src/zeroth/agent_runtime/__init__.py` imports and `__all__`: `LiteLLMProviderAdapter`, `MCPServerConfig`, `ModelParams`, `build_response_format`.
- D-06: No other symbols need adding -- scope limited to what's listed in success criteria.
- D-07: Update `.planning/phases/14-memory-connectors-container-sandbox/14-VERIFICATION.md` to reflect Phase 18 fixes.

### Claude's Discretion
- Property implementation style (single `@property` vs storing as public attr) -- either approach is fine
- Whether to add a test for the `base_url` property exposure

### Deferred Ideas (OUT OF SCOPE)
None
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| OPS-01 | Durable webhook notifications for run completion, approval needed, and failure events | The Regulus health probe fix (D-01/D-02) closes INT-03 gap against OPS-01/OPS-03. Health probe currently reports Regulus as unavailable even when reachable because `RegulusClient` lacks a `base_url` attribute. |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- Build/test commands: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- Main package: `src/zeroth/`
- Tests: `tests/`
- Must use `progress-logger` skill for tracking
- Progress tracked in root `PROGRESS.md`

## Architecture Patterns

### Fix 1: RegulusClient base_url Property

**Current state (the bug):**
```python
# src/zeroth/econ/client.py - constructor receives base_url but doesn't store it
def __init__(self, *, base_url: str = "http://localhost:8000/v1", ...) -> None:
    self._client = InstrumentationClient(base_url=base_url, ...)
    # base_url is NOT stored on self -- lost after __init__
```

**Health probe reads it (line 155):**
```python
# src/zeroth/service/health.py
regulus_base_url = getattr(regulus_client, "base_url", None)
# Always returns None -> check_regulus(None) -> "unavailable"
```

**Fix pattern (per D-01):**
```python
def __init__(self, *, base_url: str = "http://localhost:8000/v1", ...) -> None:
    self._base_url = base_url
    self._client = InstrumentationClient(base_url=base_url, ...)

@property
def base_url(self) -> str:
    return self._base_url
```

**Confidence:** HIGH -- direct code inspection confirms the bug and the fix.

### Fix 2: Docker Compose Env Vars

**Current state:** The `zeroth` service in `docker-compose.yml` has env vars for database, Redis, sandbox, dispatch, and deployment_ref but is missing Regulus configuration. The `regulus` service is defined with `REGULUS_PORT: "8080"` but the zeroth service has no way to discover it.

**Fix pattern (per D-03, D-04):** Add two env vars following the existing `ZEROTH_{SECTION}__{KEY}` double-underscore pattern:
```yaml
ZEROTH_REGULUS__ENABLED: "true"
ZEROTH_REGULUS__BASE_URL: "http://regulus:8080/v1"
```

**Placement:** After `ZEROTH_DEPLOYMENT_REF` in the zeroth service environment block, maintaining alphabetical section grouping.

**Confidence:** HIGH -- the naming pattern matches existing env vars (e.g., `ZEROTH_DATABASE__BACKEND`, `ZEROTH_SANDBOX__BACKEND`).

### Fix 3: Agent Runtime Re-exports

**Current state:** `agent_runtime/__init__.py` exports 30 symbols but is missing 4 that exist in submodules:

| Symbol | Source Module | Line |
|--------|--------------|------|
| `LiteLLMProviderAdapter` | `provider.py` | class at line 138 |
| `MCPServerConfig` | `mcp.py` | class at line 20 |
| `ModelParams` | `models.py` | class at line 67 |
| `build_response_format` | `response_format.py` | function at line 10 |

**Fix pattern (per D-05):** Add imports from submodules and insert into `__all__` alphabetically:
- `LiteLLMProviderAdapter` goes in the `provider` import block (already imports from that module)
- `MCPServerConfig` needs a new `from zeroth.agent_runtime.mcp import MCPServerConfig`
- `ModelParams` goes in the existing `models` import block
- `build_response_format` needs a new `from zeroth.agent_runtime.response_format import build_response_format`

**Confidence:** HIGH -- verified all 4 symbols exist at the expected locations.

### Fix 4: Phase 14 Verification Update

**Current state:** `14-VERIFICATION.md` has `status: gaps_found` with two gaps:
1. `ConnectorScope` ImportError in factory.py (cascading failure blocking all 12 truths)
2. MEM-06 requirement marked incomplete in REQUIREMENTS.md

**What Phase 18 fixed:** Phase 18 Plan 01 (commit `52ed53c`) wired real memory settings and completed MEM-01/MEM-06 requirements. The ConnectorScope import was resolved (verified: `factory.py` no longer references `ConnectorScope`). REQUIREMENTS.md now shows MEM-06 as complete.

**Fix pattern (per D-07):** Update the YAML frontmatter status from `gaps_found` to `passed` and add a note in the body explaining the gaps were resolved by Phase 18 cross-phase integration wiring.

**Confidence:** HIGH -- verified factory.py has no ConnectorScope references and REQUIREMENTS.md shows MEM-06 complete.

## Don't Hand-Roll

Not applicable for this phase -- all fixes are direct edits to existing files. No libraries, frameworks, or custom solutions needed.

## Common Pitfalls

### Pitfall 1: Wrong base_url default in RegulusClient property
**What goes wrong:** The RegulusClient constructor default is `http://localhost:8000/v1` but Docker Compose uses `http://regulus:8080/v1`. These are different.
**Why it happens:** The property just returns the stored constructor argument, so the default only matters when no base_url is passed.
**How to avoid:** The property stores whatever was passed to `__init__`. Docker Compose passes the correct URL via env var. No action needed beyond the basic fix.

### Pitfall 2: Import order for MCP in agent_runtime __init__
**What goes wrong:** `mcp.py` has lazy imports (MCP SDK imports inside `start()`) but `MCPServerConfig` is a Pydantic model at module level -- safe to import.
**Why it happens:** The MCP SDK is optional. But `MCPServerConfig` is a pure Pydantic BaseModel with no MCP SDK imports at module level.
**How to avoid:** Verify `MCPServerConfig` class definition only uses `pydantic` and standard library imports. Confirmed: line 20 of `mcp.py` defines it as `class MCPServerConfig(BaseModel)` with only pydantic/typing imports above.

### Pitfall 3: Forgetting to update __all__ after adding imports
**What goes wrong:** Symbols are imported but not in `__all__`, so `from zeroth.agent_runtime import *` misses them.
**How to avoid:** Add to both the import statements AND the `__all__` list. The existing `__all__` is alphabetically sorted -- maintain that order.

## Code Examples

### RegulusClient Property Addition
```python
# src/zeroth/econ/client.py
class RegulusClient:
    def __init__(
        self,
        *,
        base_url: str = "http://localhost:8000/v1",
        timeout: float = 5.0,
        enabled: bool = True,
    ) -> None:
        self._base_url = base_url
        self._client = InstrumentationClient(
            base_url=base_url,
            timeout=timeout,
            enabled=enabled,
        )

    @property
    def base_url(self) -> str:
        """Return the Regulus API base URL."""
        return self._base_url
```

### Docker Compose Env Vars Addition
```yaml
# docker-compose.yml zeroth service environment block
environment:
  ZEROTH_DATABASE__BACKEND: postgres
  ZEROTH_DATABASE__POSTGRES_DSN: "postgresql://zeroth:zeroth@postgres:5432/zeroth"
  ZEROTH_REDIS__HOST: redis
  ZEROTH_REDIS__PORT: "6379"
  ZEROTH_REGULUS__ENABLED: "true"
  ZEROTH_REGULUS__BASE_URL: "http://regulus:8080/v1"
  ZEROTH_SANDBOX__BACKEND: sidecar
  ZEROTH_SANDBOX__SIDECAR_URL: "http://sandbox-sidecar:8001"
  ZEROTH_DISPATCH__ARQ_ENABLED: "true"
  ZEROTH_DEPLOYMENT_REF: "default"
```

### Agent Runtime Re-exports Addition
```python
# Add to existing imports in agent_runtime/__init__.py
from zeroth.agent_runtime.mcp import MCPServerConfig
from zeroth.agent_runtime.models import (
    AgentConfig,
    AgentRunResult,
    InMemoryThreadStateStore,
    ModelParams,           # <-- add
    PromptAssembly,
    PromptConfig,
    PromptMessage,
    RetryPolicy,
)
from zeroth.agent_runtime.provider import (
    DeterministicProviderAdapter,
    GovernedLLMProviderAdapter,
    LiteLLMProviderAdapter,  # <-- add
    ProviderAdapter,
    ProviderMessage,
    ProviderRequest,
    ProviderResponse,
)
from zeroth.agent_runtime.response_format import build_response_format

# Add to __all__ (alphabetical positions):
# "LiteLLMProviderAdapter" after "InMemoryThreadStateStore"
# "MCPServerConfig" after "InMemoryThreadStateStore" / "LiteLLMProviderAdapter"
# "ModelParams" after "MCPServerConfig"
# "build_response_format" after "UndeclaredToolError"
```

## Testing Approach

**Existing test for RegulusClient:** `tests/test_econ_models.py` has a test for `track_execution`. A small unit test for `base_url` property would be appropriate (Claude's discretion per CONTEXT.md).

**Suggested test:**
```python
def test_regulus_client_exposes_base_url():
    from zeroth.econ.client import RegulusClient
    client = RegulusClient.__new__(RegulusClient)
    client._base_url = "http://example.com/v1"
    assert client.base_url == "http://example.com/v1"
```

**Re-export verification:** Can be tested by importing from `zeroth.agent_runtime`:
```python
from zeroth.agent_runtime import (
    LiteLLMProviderAdapter,
    MCPServerConfig,
    ModelParams,
    build_response_format,
)
```

**Linting:** Run `uv run ruff check src/` and `uv run ruff format src/` after changes.

## Open Questions

None. All four fixes are well-defined with clear success criteria and verified source locations.

## Sources

### Primary (HIGH confidence)
- Direct code inspection of `src/zeroth/econ/client.py` -- confirmed missing `base_url` attribute
- Direct code inspection of `src/zeroth/service/health.py` -- confirmed `getattr(regulus_client, "base_url", None)` at line 155
- Direct code inspection of `src/zeroth/agent_runtime/__init__.py` -- confirmed 4 missing re-exports
- Direct code inspection of `docker-compose.yml` -- confirmed missing REGULUS env vars
- Direct code inspection of `src/zeroth/memory/factory.py` -- confirmed ConnectorScope import was already fixed
- Phase 18-01 SUMMARY -- confirmed MEM-06 and memory wiring completed

## Metadata

**Confidence breakdown:**
- All fixes: HIGH -- direct source code verification, no external dependencies or uncertain APIs
- Docker Compose: HIGH -- follows established naming pattern visible in the file
- Verification update: HIGH -- Phase 18 summary confirms gap closure

**Research date:** 2026-04-09
**Valid until:** Indefinite (bug fixes against stable internal code)
