# Phase 39: Subgraph Composition - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Add subgraph composition to the orchestrator so a graph can reference another published graph as a nested node. The orchestrator enters the subgraph as a scoped execution that inherits governance, shares thread memory (configurable), and propagates approvals back to the parent.

</domain>

<decisions>
## Implementation Decisions

### Subgraph Node Model
- **D-01:** New `SubgraphNode` type (or `SubgraphNodeData` on existing node hierarchy) with fields: `graph_ref` (str — name of referenced graph), `version` (int | None — None means latest active deployment), `thread_participation` (str — "inherit" | "isolated"), `max_depth` (int, default 3 — prevents infinite recursion).
- **D-02:** Subgraph references resolved at runtime via the existing `DeploymentService` — `get_active_deployment(graph_ref)` or `get_deployment(graph_ref, version)`.

### Orchestrator Integration
- **D-03:** When the orchestrator reaches a SubgraphNode, it enters the referenced graph as a nested scope by recursively calling `_drive()` with the subgraph's graph definition. The subgraph's output maps back to the parent edge's expected input.
- **D-04:** Recursive invocation with depth tracking — each nested call increments a depth counter passed via context. If depth exceeds `max_depth`, raise `SubgraphDepthLimitError`.
- **D-05:** The subgraph creates a child Run linked to the parent via `parent_run_id`. Audit records from the child run are linked to the parent for traceability.

### Governance Inheritance
- **D-06:** Parent policies apply as a baseline — the subgraph can further restrict but not relax capabilities. Implemented by merging parent effective capabilities with subgraph policies (intersection, not union).
- **D-07:** If a HumanApprovalNode inside a subgraph pauses execution, the parent run transitions to WAITING_APPROVAL. Resolution resumes the subgraph and eventually the parent.

### Thread Memory
- **D-08:** Thread participation configurable per subgraph node: "inherit" shares the parent's thread_id (agents in subgraph see parent thread memory), "isolated" creates a new thread_id for the subgraph scope.

### Multi-Reference & Reuse
- **D-09:** The same subgraph can be referenced by multiple parent graphs and at multiple points within a single parent — each invocation creates its own child Run.
- **D-10:** Node IDs in subgraph execution are namespaced with a prefix (e.g., `subgraph:{graph_ref}:{depth}:`) to prevent collisions across nesting levels.

### Implementation Structure
- **D-11:** New `zeroth.core.subgraph` package with models.py, resolver.py, executor.py, errors.py, __init__.py.
- **D-12:** SubgraphExecutor is injected into the orchestrator at bootstrap, similar to ParallelExecutor and artifact_store patterns.

### Claude's Discretion
- SubgraphNodeData vs extending NodeBase directly
- Contract compatibility validation strategy between parent edge output and subgraph entry
- How to handle subgraph-internal state vs parent state isolation details
- Test approach for recursive invocation (mock _drive or test with real nested graphs)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Orchestrator
- `src/zeroth/core/orchestrator/runtime.py` — `_drive()` loop, `_dispatch_node()`, parallel fan-out integration from Phase 38

### Graph & Deployment
- `src/zeroth/core/graph/models.py` — Node models, NodeBase
- `src/zeroth/core/deployments/service.py` — DeploymentService — graph resolution by name/version
- `src/zeroth/core/deployments/repository.py` — Deployment persistence
- `src/zeroth/core/deployments/models.py` — Deployment model

### Runs
- `src/zeroth/core/runs/models.py` — Run model with parent_run_id capability

### Governance
- `src/zeroth/core/policy/guard.py` — PolicyGuard — governance inheritance
- `src/zeroth/core/audit/models.py` — NodeAuditRecord — subgraph audit linking

### Approvals
- `src/zeroth/core/service/approval_api.py` — Approval handling

### Bootstrap
- `src/zeroth/core/service/bootstrap.py` — Service initialization

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `_drive()` loop — Can be called recursively for subgraph execution
- `DeploymentService` — Graph resolution by name and version
- `Run` model with `parent_run_id` field — Already supports parent-child linking
- `PolicyGuard` — Policy merging/inheritance
- Phase 38's parallel execution pattern — Similar injection pattern for SubgraphExecutor

### Established Patterns
- Executor injection at bootstrap (ParallelExecutor, ArtifactStore, HttpClient, TemplateRegistry)
- `_dispatch_node()` with node type detection
- Save/restore pattern for injecting context into runner

### Integration Points
- `orchestrator/runtime.py` — SubgraphNode detection in `_drive()` or `_dispatch_node()`
- `graph/models.py` — SubgraphNodeData on the node hierarchy
- `deployments/service.py` — Graph resolution
- `service/bootstrap.py` — SubgraphExecutor initialization

</code_context>

<specifics>
## Specific Ideas

STATE.md notes: "Governance inheritance rules (parent-ceiling/child-floor) are novel — recommend research-phase." The parent-ceiling model means the child can only further restrict, never relax, the parent's policy.

</specifics>

<deferred>
## Deferred Ideas

- Subgraph runtime flattening — explicitly out of scope per REQUIREMENTS.md ("Airflow tried this and deprecated it")
- Cross-process subgraph execution — subgraphs execute in same process as parent

</deferred>

---

*Phase: 39-subgraph-composition*
*Context gathered: 2026-04-13*
