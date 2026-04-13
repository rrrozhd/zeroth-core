# Phase 40: Integration & Service Wiring - Research

**Researched:** 2026-04-12
**Domain:** Cross-feature integration validation, service bootstrap, OpenAPI, regression testing
**Confidence:** HIGH

## Summary

Phase 40 is a validation and integration phase, not a feature phase. All seven v4.0 subsystems (transform mappings, artifact store, HTTP client, templates, context window, parallel, subgraph) are individually implemented and tested. The full test suite has 1175 passing tests with zero failures. The bootstrap (`service/bootstrap.py`) already wires all subsystems, and the orchestrator (`orchestrator/runtime.py`) already integrates them into the execution loop.

The primary work is: (1) writing cross-feature integration tests that exercise subsystem interactions not covered by individual phase tests, (2) validating the OpenAPI spec reflects all v4.0 capabilities, (3) confirming backward compatibility via the full test suite, and (4) updating documentation references.

**Primary recommendation:** Focus on writing targeted cross-feature integration tests for the five interaction scenarios identified in D-05, validate the OpenAPI spec includes artifact/template endpoints if API routes exist, and run the full regression suite as a gate. The bootstrap wiring is already complete -- this phase is about validation, not construction.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Service bootstrap initializes all new subsystems (artifact store, HTTP client, template registry, context window manager) and makes them available to the orchestrator and agent runtime without manual configuration beyond settings.
- **D-02:** Each v4.0 subsystem follows the same bootstrap pattern: read settings -> construct instance -> inject into orchestrator. Phases 34-39 already wired their individual subsystems -- this phase validates the collective wiring.
- **D-03:** The OpenAPI spec includes endpoints for new v4.0 capabilities where applicable: artifact retrieval, template CRUD, parallel run status. Generated from FastAPI route decorators.
- **D-04:** No new REST endpoints needed for context window management (internal to agent runtime) or subgraph composition (internal to orchestrator). Only artifact and template subsystems expose HTTP APIs.
- **D-05:** Cross-feature interactions that must work:
  - Parallel branches can use artifact store for large outputs
  - Agent nodes inside parallel branches respect context window limits
  - Subgraph nodes inside parallel branches execute with proper governance isolation
  - Template-resolved prompts work inside parallel branches and subgraphs
  - HTTP client is available inside subgraph execution
- **D-06:** All existing tests continue to pass (backward compatibility verified by full test suite).
- **D-07:** Update existing docs to reference new v4.0 capabilities. No new doc site pages (docs site is in zeroth-studio repo).

### Claude's Discretion
- Which cross-feature scenarios to test (prioritize based on risk)
- Whether to add API endpoints for artifact and template operations or just verify existing wiring
- Test structure for cross-feature integration tests

### Deferred Ideas (OUT OF SCOPE)
None -- this is the final integration phase.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| D-01/D-02 | Bootstrap validates all subsystems initialized | Research confirms all 7 subsystems wired in bootstrap.py (lines 324-379, 434-471). Verification = code inspection + bootstrap smoke test. |
| D-03/D-04 | OpenAPI spec reflects v4.0 capabilities | Research confirms NO artifact or template HTTP routes exist yet. API routes need to be added or D-03 needs re-evaluation. |
| D-05 | Five cross-feature interaction scenarios work | Research identifies HOW each works (or fails) in the current code. See Architecture Patterns. |
| D-06 | All existing tests pass | Confirmed: 1175/1175 pass as of research date. |
| D-07 | Docs updated | Docs site is in separate repo; this phase updates in-repo docs only. |
</phase_requirements>

## Standard Stack

No new libraries needed. This phase uses the existing test infrastructure.

### Core (Existing)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pytest | existing | Test framework | Already used for all 1175 tests [VERIFIED: codebase] |
| pytest-asyncio | existing | Async test support | All orchestrator tests use it [VERIFIED: codebase] |
| httpx | existing | Mock transport for HTTP integration tests | Already used in HTTP client tests [VERIFIED: codebase] |
| FastAPI TestClient | existing | API route testing | Already used in service tests [VERIFIED: codebase] |

No `npm install` or `pip install` needed beyond the existing `uv sync`.

## Architecture Patterns

### Current Bootstrap Wiring (Already Complete)

The bootstrap in `service/bootstrap.py` already wires all v4.0 subsystems. [VERIFIED: codebase inspection]

```
ServiceBootstrap dataclass fields (v4.0 additions):
├── artifact_store (Phase 34) - line 140
├── http_client (Phase 35) - line 142
├── template_registry (Phase 36) - line 144
├── subgraph_executor (Phase 39) - line 149
└── (Context window: no field needed, orchestrator.context_window_enabled=True)

bootstrap_service() wiring:
├── Artifact store: lines 324-349 (filesystem or redis based on settings)
├── HTTP client: lines 352-364 (ResilientHttpClient from settings)
├── Template registry: lines 367-372 (TemplateRegistry + TemplateRenderer)
└── Subgraph executor: lines 375-380 (SubgraphResolver + SubgraphExecutor)
```

### Orchestrator Integration Points (Already Wired)

The `RuntimeOrchestrator` integrates all features via `_dispatch_node()` and the `_drive()` loop. [VERIFIED: codebase inspection]

```
_dispatch_node(node, run, input_payload):
├── Phase 36: Template resolution (lines 667-716) - resolves template_ref, renders with variables
├── Phase 18: Cost instrumentation (lines 728-750) - wraps provider with InstrumentedProviderAdapter
├── Phase 20: Memory resolver injection (lines 757-770)
├── Phase 37: Context window tracker injection (lines 773-801) - creates tracker from settings
└── Returns (output_data, audit_record)

_drive(graph, run) loop:
├── HumanApprovalNode → pause run (line 236)
├── SubgraphNode → delegate to SubgraphExecutor (lines 274-407)
├── AgentNode/ExecutableUnitNode → _dispatch_node() (line 410)
└── Parallel fan-out → _execute_parallel_fan_out() (lines 416-430)
```

### Cross-Feature Interaction Analysis

**This is the critical research finding.** Here is how each D-05 interaction works (or doesn't) in the current code:

#### 1. Parallel branches + artifact store (WORKS)
Parallel `branch_coro_factory` calls `self._dispatch_node(ds_node, run, branch_output)` at line 552. The `_dispatch_node` method executes the agent runner, which produces output. After fan-in, the merged output goes through `_refresh_artifact_ttls` at line 471. Artifact externalization happens at the output level. **Risk: LOW.** [VERIFIED: codebase]

#### 2. Agent nodes inside parallel branches + context window (WORKS)
`_dispatch_node` handles context window tracker injection at lines 773-801 regardless of call context. When called from `branch_coro_factory`, the same injection occurs. Each branch dispatches through `_dispatch_node` independently. **Risk: MEDIUM.** The context tracker is injected per-dispatch and restored in `finally` (line 834). But since branches run concurrently and share the same `runner` object, there is a **potential race condition** on runner attribute mutation. The `finally` block restores `runner.context_tracker = original_context_tracker`, but another branch could be mid-execution with the same runner. [VERIFIED: codebase]

#### 3. Subgraph nodes inside parallel branches (DOES NOT WORK)
The parallel `branch_coro_factory` (line 531) iterates over downstream nodes and calls `self._dispatch_node(ds_node, run, branch_output)` for each. However, `_dispatch_node` only handles `AgentNode` and `ExecutableUnitNode` -- it raises `NodeDispatcherError("unsupported node type")` for `SubgraphNode`. SubgraphNode handling is in the `_drive` loop (lines 274-407), which is NOT called by `branch_coro_factory`. **This means SubgraphNodes as downstream nodes in parallel branches will raise NodeDispatcherError.** [VERIFIED: codebase, runtime.py line 877]

**Mitigation options:**
- (a) Extend `_dispatch_node` to handle SubgraphNode (significant complexity, would need to replicate _drive loop SubgraphNode handling)
- (b) Document this as an unsupported combination and add validation in `split_fan_out` to reject SubgraphNode downstream (like HumanApprovalNode is already rejected)
- (c) Have `branch_coro_factory` use a mini-_drive loop instead of raw `_dispatch_node`

Given the complexity and risk, **option (b) is recommended** -- add a validation guard similar to the existing HumanApprovalNode guard. SubgraphNode inside a parallel branch is an edge case that could be supported in a future phase.

#### 4. Template-resolved prompts in parallel branches and subgraphs (WORKS)
Templates are resolved inside `_dispatch_node` (lines 667-716). Parallel branches call `_dispatch_node`, so templates work. Subgraphs call `_drive()` recursively, which calls `_dispatch_node` for agent nodes, so templates also work in subgraphs. **Risk: LOW** (same runner mutation concern as context window). [VERIFIED: codebase]

#### 5. HTTP client available inside subgraph execution (PARTIAL)
The `http_client` field is set on the orchestrator (line 101, `http_client: Any | None = None`). However, **it is never actually injected into agent runners or execution units** -- `_dispatch_node` does not reference `self.http_client` at all. It is stored on the orchestrator and bootstrap for access by callers who explicitly retrieve it, but not automatically injected. Since subgraph execution reuses the same orchestrator instance (via `orchestrator._drive()`), the http_client IS available on `self` inside the subgraph, but it's not being passed to runners. **Risk: LOW** -- this is the intended design (the HTTP client is for execution units to access externally, not for automatic injection). [VERIFIED: codebase]

### Critical Race Condition in Parallel Branches

**IMPORTANT FINDING:** The `_dispatch_node` method mutates `runner` attributes (provider, memory_resolver, budget_enforcer, context_tracker, config) before execution and restores them in a `finally` block (lines 826-837). When parallel branches run concurrently and share the same runner (identified by `node.node_id`), multiple branches could be mutating the same runner simultaneously.

However, reviewing the code more carefully: each branch dispatches a DIFFERENT downstream node (line 535: `for ds_node_id in downstream_node_ids`), and runners are keyed by `node.node_id` in `self.agent_runners`. Since branches all execute the SAME downstream node IDs with different payloads, they would all grab the SAME runner object.

This is a **real concurrency bug**: two branches could both mutate `runner.context_tracker` simultaneously. The try/finally restore pattern is not thread-safe (or task-safe) under `asyncio.gather`. [VERIFIED: codebase, runtime.py lines 531-602 + 773-837]

**However**, since all the runner mutations and the actual runner call happen within a single `await` (the `_dispatch_node` call), and `asyncio.gather` interleaves at await points, if the runner call itself is a single await then each branch finishes its dispatch before yielding. The risk depends on whether the runner execution itself has internal await points that could interleave with another branch's setup/teardown cycle.

**Mitigation:** Use a per-dispatch copy of the runner config, or create per-dispatch runner clones. This is a pre-existing design pattern (the restore-in-finally approach predates Phase 38), so it's working in practice for the non-parallel case. For the parallel case, the risk is real but low-probability given typical execution patterns. A test should be written to exercise this path.

### OpenAPI Spec Gap

**Current state:** The OpenAPI spec has NO artifact or template endpoints. [VERIFIED: dump_openapi.py output]

D-03 states: "The OpenAPI spec includes endpoints for new v4.0 capabilities where applicable: artifact retrieval, template CRUD, parallel run status."

**Options:**
1. Add artifact retrieval and template CRUD API routes (new files in `src/zeroth/core/service/`)
2. Verify that the OpenAPI spec generation works with the current routes and update the snapshot
3. Document that artifact/template APIs are deferred (they're internal to the orchestrator, accessed via the bootstrap object not HTTP)

D-04 says "Only artifact and template subsystems expose HTTP APIs." This implies API routes SHOULD be added. However, both subsystems are currently internal -- artifact store is used by the orchestrator for payload externalization, and templates are resolved by the orchestrator before agent execution. Neither is directly user-facing via HTTP.

**Recommendation:** Add minimal REST endpoints for artifact retrieval (GET) and template CRUD (GET/POST/PUT/DELETE), following the existing pattern in `service/run_api.py` and `service/webhook_api.py`. This aligns with D-03 and D-04.

### Test Structure for Cross-Feature Integration Tests

Recommended file: `tests/test_v4_cross_feature_integration.py` [ASSUMED]

Pattern: Follow existing integration test patterns from `tests/parallel/test_drive_integration.py` and `tests/subgraph/test_integration.py`:
- Use `sqlite_db` fixture for real database
- Create graphs with multiple v4.0 features active
- Run through `RuntimeOrchestrator.run_graph()` end-to-end
- Assert on run status, output data, audit records, and metadata

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| OpenAPI spec generation | Manual spec writing | FastAPI route decorators + `dump_openapi.py` | FastAPI auto-generates from type annotations [VERIFIED: existing pattern] |
| Test database setup | Manual SQL setup | `sqlite_db` fixture from conftest.py | Runs Alembic migrations automatically [VERIFIED: conftest.py] |
| Mock agent runners | Custom mock frameworks | `CallableProviderAdapter` + `AgentRunner` | Used consistently across all orchestrator tests [VERIFIED: test patterns] |
| HTTP client mocking | Custom transport mocks | `httpx.MockTransport` | Clean, composable, already used in HTTP tests [VERIFIED: tests/http/] |

## Common Pitfalls

### Pitfall 1: Runner Mutation Under Concurrency
**What goes wrong:** Parallel branches share the same `AgentRunner` instance (keyed by downstream node_id). `_dispatch_node` mutates runner attributes before execution and restores in finally. Under `asyncio.gather`, interleaving can corrupt runner state.
**Why it happens:** The runner-mutation-and-restore pattern was designed for sequential execution (pre-Phase 38). Parallel execution was added later without making the runner dispatch reentrant.
**How to avoid:** Write a test that verifies concurrent branches with the same downstream node produce correct results. If the test fails, implement per-dispatch runner cloning or use a lock.
**Warning signs:** Intermittent test failures where context_tracker or template_registry values are wrong; "original_provider" restored incorrectly.

### Pitfall 2: SubgraphNode in Parallel Branches
**What goes wrong:** If a graph has a parallel fan-out node whose downstream nodes include a SubgraphNode, the branch execution will raise `NodeDispatcherError` because `_dispatch_node` doesn't handle SubgraphNode.
**Why it happens:** Parallel branch execution delegates to `_dispatch_node`, but SubgraphNode is handled in the `_drive` loop, not `_dispatch_node`.
**How to avoid:** Add validation in `split_fan_out` or `branch_coro_factory` to reject SubgraphNode as a downstream node type, with a clear error message.
**Warning signs:** `NodeDispatcherError("unsupported node type: <class 'SubgraphNode'>")` during parallel execution.

### Pitfall 3: Missing API Routes for D-03
**What goes wrong:** D-03 specifies the OpenAPI spec should include artifact retrieval, template CRUD, and parallel run status endpoints, but none exist yet.
**Why it happens:** Phases 34-38 focused on internal wiring, not HTTP API surfaces.
**How to avoid:** Create API route modules (`artifact_api.py`, `template_api.py`) and register them on the v1 router in `app.py`, then regenerate the OpenAPI spec.
**Warning signs:** `dump_openapi.py` output doesn't include artifact/template/parallel paths.

### Pitfall 4: OpenAPI Spec Drift After Adding Routes
**What goes wrong:** After adding new API routes, the checked-in `openapi/zeroth-core-openapi.json` becomes stale, and the CI drift gate fails.
**Why it happens:** The `dump_openapi.py --check` gate compares the generated spec against the committed snapshot.
**How to avoid:** Always regenerate the snapshot after adding/changing routes: `uv run python scripts/dump_openapi.py --out openapi/zeroth-core-openapi.json`
**Warning signs:** CI failure on `dump_openapi.py --check`.

### Pitfall 5: Bootstrap StubNamespace Missing New Fields
**What goes wrong:** `scripts/dump_openapi.py` uses a `SimpleNamespace` stub for the bootstrap. If new API routes access bootstrap fields that aren't on the stub, `app.openapi()` will fail.
**Why it happens:** The stub is minimal -- it only has fields needed at import-time, not request-time. New routes may access `bootstrap.artifact_store` or `bootstrap.template_registry` at schema-generation time (unlikely but possible).
**How to avoid:** After adding routes, test that `dump_openapi.py` still works. If it fails, add the missing fields to the stub.
**Warning signs:** `AttributeError` when running `dump_openapi.py`.

## Code Examples

### Bootstrap Validation Pattern
```python
# Source: tests/service/test_app.py (existing pattern, adapted)
# Verify all v4.0 subsystems are present on ServiceBootstrap
async def test_bootstrap_has_all_v4_subsystems(sqlite_db):
    from zeroth.core.service.bootstrap import bootstrap_service, ServiceBootstrap
    # ... setup deployment ...
    svc = await bootstrap_service(sqlite_db, deployment_ref="test-ref")
    assert svc.artifact_store is not None
    assert svc.http_client is not None
    assert svc.template_registry is not None
    assert svc.subgraph_executor is not None
    # Context window is orchestrator-level, not bootstrap-level
    assert svc.orchestrator.context_window_enabled is True
```

### Cross-Feature Integration Test Pattern
```python
# Source: tests/parallel/test_drive_integration.py (existing pattern, adapted)
# Test: Parallel branches with template resolution + context window
@pytest.mark.asyncio
async def test_parallel_branches_with_templates(sqlite_db):
    from zeroth.core.templates import TemplateRegistry, TemplateRenderer, PromptTemplate
    
    registry = TemplateRegistry()
    registry.register(PromptTemplate(name="branch-prompt", version="1", body="Process: {{input.x}}"))
    renderer = TemplateRenderer()
    
    orchestrator = RuntimeOrchestrator(
        run_repository=RunRepository(sqlite_db),
        agent_runners={"source": source_runner, "sink": sink_runner},
        executable_unit_runner=ExecutableUnitRunner(),
        template_registry=registry,
        template_renderer=renderer,
        context_window_enabled=True,
    )
    
    run = await orchestrator.run_graph(graph, {"value": 1})
    assert run.status is RunStatus.COMPLETED
```

### API Route Registration Pattern
```python
# Source: src/zeroth/core/service/webhook_api.py (existing pattern)
# Pattern for adding new v4.0 API routes
def register_artifact_routes(router: APIRouter) -> None:
    @router.get("/artifacts/{artifact_id}")
    async def get_artifact(artifact_id: str, request: Request):
        bootstrap = request.app.state.bootstrap
        store = getattr(bootstrap, "artifact_store", None)
        if store is None:
            raise HTTPException(status_code=503, detail="artifact store not configured")
        # ... retrieve and return artifact ...
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual bootstrap wiring per phase | Each phase self-wires in bootstrap_service() | Phase 34+ | No central integration step needed for wiring |
| Sequential-only node dispatch | Parallel fan-out via _dispatch_node delegation | Phase 38 | Cross-feature behavior inherited automatically |
| Single-level orchestration | Recursive _drive() for subgraphs | Phase 39 | All orchestrator features available in subgraphs |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Test file should be named `tests/test_v4_cross_feature_integration.py` | Test Structure | Low -- naming is flexible |
| A2 | Runner mutation under concurrency is low-probability in practice | Pitfall 1 | Medium -- could cause intermittent failures if runners have internal await points |
| A3 | Option (b) for SubgraphNode-in-parallel-branches is the right approach | Cross-Feature Analysis | Medium -- if users need subgraph composition inside parallel branches, a more complex solution is needed |
| A4 | Artifact and template REST endpoints should be added per D-03 | OpenAPI Spec Gap | Medium -- user may decide these are internal-only and skip HTTP exposure |

## Open Questions

1. **Should SubgraphNode inside parallel branches be supported or blocked?**
   - What we know: Currently raises `NodeDispatcherError`. The architecture doesn't support it without significant rework.
   - What's unclear: Whether this is a real user need or a theoretical edge case.
   - Recommendation: Block with a clear validation error (option b). Document as a known limitation. Support in a future phase if needed.

2. **What artifact/template API endpoints should be added?**
   - What we know: D-03 says they should exist. No routes exist today.
   - What's unclear: The exact endpoint shapes -- CRUD? Read-only? What auth scopes?
   - Recommendation: Add minimal read endpoints for artifacts (GET by ID) and basic CRUD for templates. Follow existing patterns in `webhook_api.py` and `run_api.py`.

3. **Does the runner mutation race condition need fixing in this phase?**
   - What we know: The pattern predates Phase 38 and works for sequential execution. Parallel branches share runners.
   - What's unclear: Whether real workloads trigger the interleaving scenario.
   - Recommendation: Write a targeted test. If it passes under stress, document the risk and defer the fix. If it fails, implement per-dispatch runner cloning.

## Environment Availability

Step 2.6: SKIPPED (no external dependencies identified -- this phase is code/config-only changes using existing test infrastructure).

## Sources

### Primary (HIGH confidence)
- `src/zeroth/core/service/bootstrap.py` -- full bootstrap wiring code, all v4.0 subsystems confirmed wired
- `src/zeroth/core/orchestrator/runtime.py` -- orchestrator integration, _dispatch_node, _drive loop, parallel fan-out
- `src/zeroth/core/parallel/executor.py` -- parallel executor implementation
- `src/zeroth/core/subgraph/executor.py` -- subgraph executor implementation
- `src/zeroth/core/service/app.py` -- FastAPI app factory, route registration
- `tests/parallel/test_drive_integration.py` -- existing parallel integration tests
- `tests/subgraph/test_integration.py` -- existing subgraph integration tests
- `scripts/dump_openapi.py` -- OpenAPI spec generation

### Secondary (MEDIUM confidence)
- Phase 18 verification report -- prior integration wiring phase patterns
- Phase 38 verification report -- parallel execution verification
- Phase 39 verification report -- subgraph composition verification

## Metadata

**Confidence breakdown:**
- Bootstrap wiring: HIGH -- all code inspected, every subsystem confirmed wired
- Cross-feature interactions: HIGH -- each D-05 scenario traced through source code
- OpenAPI gap: HIGH -- endpoint list verified via dump_openapi.py
- Runner concurrency risk: MEDIUM -- theoretical analysis, not empirically tested
- Test structure: MEDIUM -- based on existing patterns, not yet validated

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable -- integration phase, no external dependencies)
