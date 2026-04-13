# Phase 40: Integration & Service Wiring - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire all v4.0 features into the service bootstrap, verify cross-feature interactions work correctly, update the OpenAPI spec to reflect new capabilities, and ensure all existing tests continue to pass. This is a validation and integration phase, not a feature phase.

</domain>

<decisions>
## Implementation Decisions

### Bootstrap Wiring
- **D-01:** Service bootstrap initializes all new subsystems (artifact store, HTTP client, template registry, context window manager) and makes them available to the orchestrator and agent runtime without manual configuration beyond settings.
- **D-02:** Each v4.0 subsystem follows the same bootstrap pattern: read settings -> construct instance -> inject into orchestrator. Phases 34-39 already wired their individual subsystems — this phase validates the collective wiring.

### OpenAPI Spec
- **D-03:** The OpenAPI spec includes endpoints for new v4.0 capabilities where applicable: artifact retrieval, template CRUD, parallel run status. Generated from FastAPI route decorators.
- **D-04:** No new REST endpoints needed for context window management (internal to agent runtime) or subgraph composition (internal to orchestrator). Only artifact and template subsystems expose HTTP APIs.

### Cross-Feature Testing
- **D-05:** Cross-feature interactions that must work:
  - Parallel branches can use artifact store for large outputs
  - Agent nodes inside parallel branches respect context window limits
  - Subgraph nodes inside parallel branches execute with proper governance isolation
  - Template-resolved prompts work inside parallel branches and subgraphs
  - HTTP client is available inside subgraph execution
- **D-06:** All existing tests continue to pass (backward compatibility verified by full test suite).

### Documentation
- **D-07:** Update existing docs to reference new v4.0 capabilities. No new doc site pages (docs site is in zeroth-studio repo).

### Claude's Discretion
- Which cross-feature scenarios to test (prioritize based on risk)
- Whether to add API endpoints for artifact and template operations or just verify existing wiring
- Test structure for cross-feature integration tests

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Bootstrap
- `src/zeroth/core/service/bootstrap.py` — Central service wiring — verify all subsystems present

### OpenAPI / FastAPI
- `src/zeroth/core/service/app.py` — FastAPI application with router includes
- `src/zeroth/core/service/run_api.py` — Run management endpoints

### All v4.0 Subsystem Packages
- `src/zeroth/core/mappings/` — Phase 33: Transform mappings
- `src/zeroth/core/artifacts/` — Phase 34: Artifact store
- `src/zeroth/core/http/` — Phase 35: Resilient HTTP client
- `src/zeroth/core/templates/` — Phase 36: Prompt templates
- `src/zeroth/core/context_window/` — Phase 37: Context window management
- `src/zeroth/core/parallel/` — Phase 38: Parallel fan-out/fan-in
- `src/zeroth/core/subgraph/` — Phase 39: Subgraph composition

### Orchestrator
- `src/zeroth/core/orchestrator/runtime.py` — Central orchestration with all v4.0 integrations

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- All 7 v4.0 subsystems already individually wired and tested
- Full test suite at 1175 tests with zero failures
- Bootstrap already initializes most subsystems from prior phases

### Integration Points
- `service/bootstrap.py` — Validate all subsystems initialized
- `service/app.py` — API routes for artifact/template operations
- `orchestrator/runtime.py` — Cross-feature interactions at runtime
- Full test suite regression

</code_context>

<specifics>
## Specific Ideas

This phase is primarily about validation and ensuring everything works together. The heavy lifting was done in phases 33-39.

</specifics>

<deferred>
## Deferred Ideas

None — this is the final integration phase.

</deferred>

---

*Phase: 40-integration-service-wiring*
*Context gathered: 2026-04-13*
