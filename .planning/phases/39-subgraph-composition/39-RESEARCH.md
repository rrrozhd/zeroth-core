# Phase 39: Subgraph Composition - Research

**Researched:** 2026-04-12
**Domain:** Recursive graph orchestration, governance inheritance, thread sharing, approval propagation
**Confidence:** HIGH

## Summary

Phase 39 adds subgraph composition to the Zeroth orchestrator. A graph can reference another published graph as a nested `SubgraphNode`. When the orchestrator encounters this node, it resolves the referenced graph via `DeploymentService`, creates a child `Run` linked to the parent via a new `parent_run_id` field, and recursively calls `_drive()` with the subgraph's graph definition. The child run inherits governance policies (parent-ceiling/child-floor intersection model), optionally shares the parent's thread memory, and propagates approval pauses back up to the parent run.

This phase has four distinct engineering challenges of decreasing difficulty:

1. **Approval propagation** (HARD): When a `HumanApprovalNode` inside a subgraph pauses execution, the child run returns with `WAITING_APPROVAL` status. The parent run must also transition to `WAITING_APPROVAL`. When the approval is resolved, the child run resumes first, then the parent. This creates a multi-run resume chain that the current `resume_graph` does not handle. The child run's approval record must carry enough context (the parent_run_id chain) for the approval API to trigger cascading resumption.

2. **Governance inheritance** (MEDIUM): The parent graph's effective policies must act as a ceiling -- the subgraph can add more restrictive policies (further deny capabilities) but cannot relax what the parent denies. The current `PolicyGuard._allowed_capabilities` already uses set intersection across multiple policies, which is exactly the right primitive. The subgraph executor must collect the parent's effective policy bindings and prepend them to the subgraph's own bindings before evaluation.

3. **Recursive _drive() invocation** (MEDIUM): The orchestrator's `_drive()` method is a stateful loop that modifies a `Run` object. Calling it recursively with a separate child Run and a different Graph is safe as long as the child Run is a distinct object (not a reference to the parent). This is guaranteed because `run_repository.create()` produces a new persisted Run. Depth tracking via a counter passed through `Run.metadata["subgraph_depth"]` prevents infinite recursion.

4. **SubgraphNode type** (LOW): Adding a new node type to the Pydantic v2 discriminated union is straightforward -- define `SubgraphNodeData`, `SubgraphNode(NodeBase)` with `node_type: Literal["subgraph"]`, and extend the `Node` union. The orchestrator's `_drive()` loop handles the new type before `_dispatch_node()` (similar to how `HumanApprovalNode` is handled before dispatch).

**Primary recommendation:** Create `zeroth.core.subgraph` package following the Phase 38 pattern (models.py, resolver.py, executor.py, errors.py, __init__.py). The `SubgraphExecutor` orchestrates the full lifecycle: resolve graph, create child Run, merge parent policies, call `_drive()`, handle approval propagation. It is injected into `RuntimeOrchestrator` as a default-constructed field, matching the `parallel_executor` pattern.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** New `SubgraphNode` type (or `SubgraphNodeData` on existing node hierarchy) with fields: `graph_ref` (str -- name of referenced graph), `version` (int | None -- None means latest active deployment), `thread_participation` (str -- "inherit" | "isolated"), `max_depth` (int, default 3 -- prevents infinite recursion).
- **D-02:** Subgraph references resolved at runtime via the existing `DeploymentService` -- `get_active_deployment(graph_ref)` or `get_deployment(graph_ref, version)`.
- **D-03:** When the orchestrator reaches a SubgraphNode, it enters the referenced graph as a nested scope by recursively calling `_drive()` with the subgraph's graph definition. The subgraph's output maps back to the parent edge's expected input.
- **D-04:** Recursive invocation with depth tracking -- each nested call increments a depth counter passed via context. If depth exceeds `max_depth`, raise `SubgraphDepthLimitError`.
- **D-05:** The subgraph creates a child Run linked to the parent via `parent_run_id`. Audit records from the child run are linked to the parent for traceability.
- **D-06:** Parent policies apply as a baseline -- the subgraph can further restrict but not relax capabilities. Implemented by merging parent effective capabilities with subgraph policies (intersection, not union).
- **D-07:** If a HumanApprovalNode inside a subgraph pauses execution, the parent run transitions to WAITING_APPROVAL. Resolution resumes the subgraph and eventually the parent.
- **D-08:** Thread participation configurable per subgraph node: "inherit" shares the parent's thread_id (agents in subgraph see parent thread memory), "isolated" creates a new thread_id for the subgraph scope.
- **D-09:** The same subgraph can be referenced by multiple parent graphs and at multiple points within a single parent -- each invocation creates its own child Run.
- **D-10:** Node IDs in subgraph execution are namespaced with a prefix (e.g., `subgraph:{graph_ref}:{depth}:`) to prevent collisions across nesting levels.
- **D-11:** New `zeroth.core.subgraph` package with models.py, resolver.py, executor.py, errors.py, __init__.py.
- **D-12:** SubgraphExecutor is injected into the orchestrator at bootstrap, similar to ParallelExecutor and artifact_store patterns.

### Claude's Discretion
- SubgraphNodeData vs extending NodeBase directly
- Contract compatibility validation strategy between parent edge output and subgraph entry
- How to handle subgraph-internal state vs parent state isolation details
- Test approach for recursive invocation (mock _drive or test with real nested graphs)

### Deferred Ideas (OUT OF SCOPE)
- Subgraph runtime flattening -- explicitly out of scope per REQUIREMENTS.md ("Airflow tried this and deprecated it")
- Cross-process subgraph execution -- subgraphs execute in same process as parent
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SUBG-01 | A graph author can reference another published graph as a SubgraphNode, specifying graph_ref, optional version, thread_participation mode, and max_depth | SubgraphNodeData model with Pydantic discriminated union extension; DeploymentService.get() for runtime resolution |
| SUBG-02 | The orchestrator resolves the subgraph reference at runtime and executes it as a nested scope via recursive _drive() | SubgraphExecutor calls DeploymentService.get(graph_ref, version) to load the deployment, deserialize_graph() to get Graph, then orchestrator._drive(subgraph, child_run) |
| SUBG-03 | A child Run is created for each subgraph invocation, linked to the parent via parent_run_id | New parent_run_id field on Run model (currently absent -- must be added); child Run created with parent_run_id set |
| SUBG-04 | Governance policies from the parent graph act as a ceiling -- the subgraph can restrict but not relax | PolicyGuard._allowed_capabilities already computes intersection; parent policy_bindings prepended to subgraph policy_bindings before evaluation |
| SUBG-05 | Thread participation is configurable: "inherit" shares parent thread_id, "isolated" creates a new one | Child Run's thread_id set to parent's thread_id (inherit) or generated fresh (isolated) based on SubgraphNodeData.thread_participation |
| SUBG-06 | Approval pauses inside a subgraph propagate to the parent run (WAITING_APPROVAL), and resolution cascades back down | Child run returns WAITING_APPROVAL; parent detects this and transitions to WAITING_APPROVAL with metadata linking to child_run_id; resume_graph resolves child first, then continues parent |
| SUBG-07 | Depth tracking prevents infinite recursion, raising SubgraphDepthLimitError when max_depth is exceeded | Depth counter in Run.metadata["subgraph_depth"]; incremented per nesting level; checked before entering subgraph |
| SUBG-08 | Node IDs in subgraph execution are namespaced to prevent collisions across nesting levels | Prefix strategy `subgraph:{graph_ref}:{depth}:` applied to node_ids in the resolved subgraph before execution |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Build/Test:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Layout:** Source in `src/zeroth/`, tests in `tests/`
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill
- **Context efficiency:** Read only task-relevant files; do NOT read root PLAN.md
- **Pydantic pattern:** `ConfigDict(extra="forbid")` on all models [VERIFIED: codebase inspection -- every model in graph/models.py, runs/models.py, policy/models.py, parallel/models.py]
- **Package pattern:** models.py, errors.py, executor/service module, __init__.py with explicit re-exports [VERIFIED: parallel package structure]
- **Dataclass for executors/services:** `@dataclass(slots=True)` for runtime services (RuntimeOrchestrator, DeploymentService, ParallelExecutor) [VERIFIED: codebase inspection]

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pydantic | 2.12.5 | SubgraphNodeData, SubgraphConfig models, Node union extension | Project standard; ConfigDict(extra="forbid") pattern [VERIFIED: `uv run python -c "import pydantic; print(pydantic.__version__)"` -> 2.12.5] |
| asyncio (stdlib) | Python 3.12.12 | Recursive _drive() is async; approval propagation uses async run_repository | Already used throughout codebase [VERIFIED: runtime.py imports] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| copy (stdlib) | Python 3.12.12 | Deep-copy parent policy state for child governance isolation | When creating merged policy bindings for the child scope |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Recursive `_drive()` | Separate SubgraphOrchestrator class | Duplicates 1300 lines of `_drive()` loop logic; recursive call reuses all existing dispatch, policy, audit, and approval infrastructure with zero duplication |
| `parent_run_id` on Run model | `Run.metadata["parent_run_id"]` | Metadata dict is untyped; a proper field enables queries, foreign-key-style lookups, and type safety. The field addition is trivial (one line). |
| Node ID namespacing via prefix | Separate namespace scope object | Prefix is simpler, deterministic, and visible in audit trails. A scope object adds indirection with no benefit for same-process execution. |

**Installation:**
```bash
# No new packages needed -- all dependencies already installed
uv sync
```

**Version verification:** Python 3.12.12 confirmed. Pydantic 2.12.5 confirmed. No new dependencies. [VERIFIED: runtime checks]

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/core/subgraph/
    __init__.py          # Public API re-exports
    models.py            # SubgraphNodeData, SubgraphConfig (depth, thread mode)
    resolver.py          # SubgraphResolver -- resolves graph_ref to Graph via DeploymentService
    executor.py          # SubgraphExecutor -- creates child Run, merges governance, calls _drive()
    errors.py            # SubgraphDepthLimitError, SubgraphResolutionError, SubgraphExecutionError
```

### Pattern 1: SubgraphNode Type in Discriminated Union
**What:** Add `SubgraphNode(NodeBase)` with `node_type: Literal["subgraph"] = "subgraph"` and a `subgraph: SubgraphNodeData` field, then extend the `Node` union.
**When to use:** Always -- this is the data model surface per D-01.
**Example:**
```python
# Source: Derived from existing AgentNode/ExecutableUnitNode/HumanApprovalNode pattern
# in graph/models.py (lines 163-247)

class SubgraphNodeData(BaseModel):
    """Configuration for a subgraph composition step."""
    model_config = ConfigDict(extra="forbid")

    graph_ref: str
    """Name/ref of the published graph to invoke as a subgraph."""

    version: int | None = None
    """Specific deployment version. None means latest active."""

    thread_participation: Literal["inherit", "isolated"] = "inherit"
    """Thread sharing mode: 'inherit' shares parent thread_id,
    'isolated' creates a fresh thread for the subgraph."""

    max_depth: int = Field(default=3, ge=1, le=10)
    """Maximum nesting depth. Prevents infinite recursion."""


class SubgraphNode(NodeBase):
    """A graph node that invokes another published graph as a nested scope."""
    node_type: Literal["subgraph"] = "subgraph"
    subgraph: SubgraphNodeData

    def to_governed_step_spec(self) -> GovernedStepSpec:
        return GovernedStepSpec(
            name=self.node_id,
            agent={
                "kind": "subgraph_ref",
                "graph_ref": self.subgraph.graph_ref,
                "version": self.subgraph.version,
            },
        )


# Update the Node union:
Node = Annotated[
    AgentNode | ExecutableUnitNode | HumanApprovalNode | SubgraphNode,
    Field(discriminator="node_type"),
]
```
[VERIFIED: Pydantic v2 discriminated unions support adding new variants by simply extending the Annotated union type]

### Pattern 2: SubgraphResolver -- Graph Resolution at Runtime
**What:** A thin service that resolves `graph_ref` + optional `version` to a fully deserialized `Graph` object using the existing `DeploymentService`.
**When to use:** Called by `SubgraphExecutor` before entering the subgraph.
**Example:**
```python
# Source: DeploymentService.get() returns Deployment with serialized_graph field
# (deployments/service.py line 100-102, deployments/models.py line 35)

@dataclass(slots=True)
class SubgraphResolver:
    """Resolves subgraph references to Graph objects at runtime."""

    deployment_service: DeploymentService

    async def resolve(
        self,
        graph_ref: str,
        version: int | None = None,
    ) -> tuple[Graph, Deployment]:
        """Resolve a graph_ref to a Graph and its Deployment.

        Args:
            graph_ref: The deployment_ref of the published graph.
            version: Specific deployment version, or None for latest active.

        Returns:
            Tuple of (Graph, Deployment).

        Raises:
            SubgraphResolutionError: If the graph_ref cannot be resolved.
        """
        deployment = await self.deployment_service.get(graph_ref, version)
        if deployment is None:
            raise SubgraphResolutionError(
                f"subgraph reference '{graph_ref}' "
                f"{'version ' + str(version) + ' ' if version else ''}"
                f"not found"
            )
        graph = deserialize_graph(deployment.serialized_graph)
        return graph, deployment
```
**Key insight:** The CONTEXT.md says `get_active_deployment(graph_ref)` but this method does not exist. The actual API is `DeploymentService.get(deployment_ref, version)` where `deployment_ref` is the string key (which maps to `graph_ref` in the subgraph context). Using `version=None` returns the latest active deployment. No new DeploymentService methods are needed. [VERIFIED: deployments/service.py line 100-102]

### Pattern 3: Child Run Creation with parent_run_id
**What:** Create a new Run for the subgraph execution, linked to the parent via `parent_run_id`. The child inherits tenant_id and optionally thread_id from the parent.
**When to use:** Every time a SubgraphNode is entered.
**Example:**
```python
# Source: Derived from RuntimeOrchestrator.run_graph() (runtime.py lines 118-131)

async def _create_child_run(
    self,
    orchestrator: RuntimeOrchestrator,
    parent_run: Run,
    subgraph: Graph,
    subgraph_node: SubgraphNode,
    input_payload: dict[str, Any],
    depth: int,
) -> Run:
    """Create a child Run for subgraph execution."""
    thread_id = (
        parent_run.thread_id
        if subgraph_node.subgraph.thread_participation == "inherit"
        else ""  # Empty string triggers auto-generation via _fill_governai_defaults
    )
    child_run = Run(
        graph_version_ref=f"{subgraph.graph_id}:v{subgraph.version}",
        deployment_ref=subgraph_node.subgraph.graph_ref,
        tenant_id=parent_run.tenant_id,
        workspace_id=parent_run.workspace_id,
        thread_id=thread_id,
        parent_run_id=parent_run.run_id,  # NEW FIELD
        current_node_ids=[],
        pending_node_ids=[orchestrator._entry_step(subgraph)],
        metadata={
            **orchestrator._initial_metadata(subgraph, input_payload),
            "subgraph_depth": depth,
            "parent_run_id": parent_run.run_id,
            "parent_node_id": subgraph_node.node_id,
        },
    )
    persisted = await orchestrator.run_repository.create(child_run)
    persisted.status = RunStatus.RUNNING
    persisted.touch()
    return await orchestrator.run_repository.put(persisted)
```

### Pattern 4: Governance Inheritance (Parent-Ceiling / Child-Floor)
**What:** The parent's effective policies act as a ceiling. The subgraph can further restrict but not relax. Implemented by prepending the parent graph's policy_bindings to the subgraph's policy_bindings before PolicyGuard evaluation.
**When to use:** When entering a subgraph scope.
**Example:**
```python
# Source: PolicyGuard._allowed_capabilities() (policy/guard.py lines 117-136)
# already computes set INTERSECTION across multiple policies.
# Prepending parent policies means the intersection includes parent restrictions.

def _merge_governance(
    parent_graph: Graph,
    subgraph: Graph,
) -> Graph:
    """Create a governance-merged copy of the subgraph.

    Prepends parent graph's policy_bindings so the PolicyGuard's
    intersection logic automatically enforces parent-ceiling semantics.
    Parent denials cannot be overridden by the subgraph.
    """
    merged_policy_bindings = (
        list(parent_graph.policy_bindings) + list(subgraph.policy_bindings)
    )
    return subgraph.model_copy(
        update={"policy_bindings": merged_policy_bindings}
    )
```
**Why this works:** The `PolicyGuard.evaluate()` method collects policies from `graph.policy_bindings` and `node.policy_bindings`, then computes `_allowed_capabilities` as the intersection of all allow-lists and the union of all deny-lists. By prepending the parent's policy bindings, any capability the parent denies remains denied in the subgraph (because denied capabilities are unioned across all policies). Any capability the parent allows but the subgraph denies is also denied (because the deny list takes precedence). This is exactly the parent-ceiling/child-floor model described in D-06. [VERIFIED: policy/guard.py lines 56-115]

### Pattern 5: Subgraph Detection in _drive() Loop
**What:** Detect `SubgraphNode` in the main `_drive()` loop BEFORE `_dispatch_node()` is called, similar to how `HumanApprovalNode` is handled (line 226). SubgraphNode is NOT dispatched via `_dispatch_node()` -- it is handled by `SubgraphExecutor`.
**When to use:** This is the single integration point in the orchestrator's `_drive()` loop.
**Example:**
```python
# Source: _drive() loop structure (runtime.py lines 171-325)
# Insert AFTER the HumanApprovalNode check (line 261) and BEFORE _dispatch_node (line 263)

from zeroth.core.graph import SubgraphNode  # Added to imports

# Inside _drive() while True loop, after side-effect gate check:
if isinstance(node, SubgraphNode):
    try:
        child_run = await self._execute_subgraph(
            graph, run, node, node_id, input_payload,
        )
    except (SubgraphDepthLimitError, SubgraphResolutionError, SubgraphExecutionError) as exc:
        return await self._fail_run(run, "subgraph_execution_failed", str(exc))

    if child_run.status == RunStatus.WAITING_APPROVAL:
        # Propagate approval pause to parent run (D-07)
        run.status = RunStatus.WAITING_APPROVAL
        run.metadata["pending_subgraph"] = {
            "child_run_id": child_run.run_id,
            "node_id": node_id,
            "graph_ref": node.subgraph.graph_ref,
        }
        run.pending_node_ids.insert(0, node_id)  # Re-queue for resume
        run.touch()
        persisted = await self.run_repository.put(run)
        await self.run_repository.write_checkpoint(persisted)
        return persisted

    # Subgraph completed -- use its final_output as this node's output
    output_data = child_run.final_output or {}
    audit_record = {
        "subgraph_run_id": child_run.run_id,
        "subgraph_graph_ref": node.subgraph.graph_ref,
        "subgraph_status": child_run.status.value,
    }
    # Continue normal flow: record history, plan next, queue next
    await self._record_history(run, node, node_id, input_payload, output_data, audit_record)
    self._increment_node_visit(run, node_id)
    next_node_ids = self._plan_next_nodes(graph, run, node_id, output_data)
    self._queue_next_nodes(graph, run, node_id, output_data, next_node_ids)
    run.metadata["last_output"] = output_data
    # ... checkpoint and continue
    continue
```

### Pattern 6: Approval Propagation (Parent-Child Resume Chain)
**What:** When a subgraph's child run pauses for approval, the parent run also pauses. When the approval is resolved, the parent run is resumed, which must detect the pending subgraph and resume the child first.
**When to use:** Every time a subgraph contains a HumanApprovalNode.
**Example:**
```python
# Source: Derived from resume_graph() (runtime.py lines 133-145) and
# _consume_side_effect_approval (the existing approval resume pattern)

# In _drive() when re-entering a SubgraphNode after resume:
if isinstance(node, SubgraphNode):
    pending_subgraph = run.metadata.get("pending_subgraph")
    if pending_subgraph and pending_subgraph["node_id"] == node_id:
        # Resuming after approval -- continue the child run
        child_run_id = pending_subgraph["child_run_id"]
        subgraph, _ = await self.subgraph_executor.resolver.resolve(
            node.subgraph.graph_ref, node.subgraph.version,
        )
        subgraph = self.subgraph_executor._merge_governance(graph, subgraph)
        child_run = await self.resume_graph(subgraph, child_run_id)

        if child_run.status == RunStatus.WAITING_APPROVAL:
            # Still waiting -- stay paused
            return run  # Already in WAITING_APPROVAL state

        # Child completed -- clear pending_subgraph, continue parent
        del run.metadata["pending_subgraph"]
        output_data = child_run.final_output or {}
        # ... proceed with normal post-node flow
```

### Pattern 7: Node ID Namespacing
**What:** Before executing a subgraph, prefix all node_ids in the resolved Graph to prevent collisions. The prefix includes the graph_ref and depth level.
**When to use:** Before calling _drive() with the subgraph.
**Example:**
```python
# Source: D-10 from CONTEXT.md

def _namespace_subgraph(
    graph: Graph,
    graph_ref: str,
    depth: int,
) -> Graph:
    """Namespace all node and edge IDs in a subgraph to prevent collisions."""
    prefix = f"subgraph:{graph_ref}:{depth}:"
    namespaced_nodes = []
    for node in graph.nodes:
        node_copy = node.model_copy(
            update={"node_id": f"{prefix}{node.node_id}"}
        )
        namespaced_nodes.append(node_copy)

    namespaced_edges = []
    for edge in graph.edges:
        edge_copy = edge.model_copy(update={
            "edge_id": f"{prefix}{edge.edge_id}",
            "source_node_id": f"{prefix}{edge.source_node_id}",
            "target_node_id": f"{prefix}{edge.target_node_id}",
        })
        namespaced_edges.append(edge_copy)

    entry_step = f"{prefix}{graph.entry_step}" if graph.entry_step else None
    return graph.model_copy(update={
        "nodes": namespaced_nodes,
        "edges": namespaced_edges,
        "entry_step": entry_step,
    })
```

### Anti-Patterns to Avoid
- **Flattening the subgraph into the parent graph:** Airflow tried this with SubDagOperator and deprecated it in favor of TaskGroup. Flattening breaks governance isolation, makes debugging impossible, and creates node ID collision nightmares. Subgraphs MUST remain separate execution scopes with their own Runs. [CITED: CONTEXT.md deferred item; Airflow SubDagOperator deprecation in Airflow 2.0]
- **Calling _dispatch_node() for SubgraphNode:** SubgraphNode is not an agent or executable unit. It should not go through `_dispatch_node()`. Handle it in the `_drive()` loop directly, like `HumanApprovalNode`.
- **Sharing the parent Run object with the child:** The child must have its own Run instance. Sharing the parent Run would corrupt state, especially execution_history, node_visit_counts, and pending_node_ids.
- **Modifying DeploymentService for subgraph-specific resolution:** The existing `DeploymentService.get(deployment_ref, version)` already does what's needed. `deployment_ref` maps to `graph_ref`. No new methods required.
- **Deep-copying the entire orchestrator for the child scope:** The RuntimeOrchestrator is stateless between _drive() calls (all mutable state is in Run). The same orchestrator instance can call _drive() recursively with different Graph and Run objects.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Graph resolution by name/version | Custom graph loader | `DeploymentService.get(graph_ref, version)` + `deserialize_graph()` | Already handles version resolution, lineage validation, and attestation verification [VERIFIED: deployments/service.py] |
| Policy inheritance merge | Custom capability merging logic | Prepend parent policy_bindings + existing `PolicyGuard._allowed_capabilities()` intersection | PolicyGuard already computes intersection across multiple policies [VERIFIED: policy/guard.py lines 117-136] |
| Child Run creation | Manual Run construction | Follow `RuntimeOrchestrator.run_graph()` pattern (runtime.py lines 118-131) | Consistent with existing run lifecycle, triggers same validator hooks, persistence patterns |
| Node dispatch for subgraph agents | Duplicate _dispatch_node() | Reuse the same orchestrator's `_drive()` recursively | _drive() already handles all node types, audit, policy, approvals -- reusing it means zero duplication |
| Approval resume chain | Custom approval handler | Extend existing `resume_graph()` to detect pending_subgraph metadata | Existing approval infrastructure (ApprovalService, ApprovalRecord, webhook emit) already works |
| Graph serialization/deserialization | Custom JSON handling | `serialize_graph()` / `deserialize_graph()` from `graph/serialization.py` | Already tested, handles all node types via Pydantic's discriminated union |

**Key insight:** The subgraph feature reuses MORE existing infrastructure than any previous phase. The orchestrator, deployment service, policy guard, audit system, approval service, and run repository all work unchanged. The new code is primarily glue: resolving references, creating child Runs, and handling the approval propagation edge case.

## Common Pitfalls

### Pitfall 1: parent_run_id Field Does Not Exist Yet
**What goes wrong:** The CONTEXT.md claims "Run model with parent_run_id field -- Already supports parent-child linking" but this field does NOT exist on the Run model or GovernAI's RunState.
**Why it happens:** The discuss-phase assumed the field existed based on the feature name.
**How to avoid:** Add `parent_run_id: str | None = None` to the `Run` model in `runs/models.py`. This is a one-line addition. The field is optional (None for non-subgraph runs), so all existing code and tests continue to work.
**Warning signs:** `Run(parent_run_id="abc")` raises `extra fields not permitted` from `ConfigDict(extra="forbid")`.

### Pitfall 2: Circular Subgraph References
**What goes wrong:** Graph A references Graph B as a subgraph, and Graph B references Graph A. When A runs, it enters B, which tries to enter A, creating infinite recursion that only stops when max_depth is exceeded.
**Why it happens:** No validation prevents circular references at graph authoring time.
**How to avoid:** The `max_depth` field (D-01, default 3) is the primary defense. When depth exceeds max_depth, `SubgraphDepthLimitError` is raised immediately. For defense-in-depth, the executor should also track visited graph_refs in the recursion chain and detect cycles early (before reaching max_depth). This is a simple set check: `if graph_ref in visited_refs: raise SubgraphCycleError`.
**Warning signs:** Runs consistently failing with `SubgraphDepthLimitError` when the graph should only be 1-2 levels deep.

### Pitfall 3: Approval Propagation Resume Chain Failure
**What goes wrong:** An approval inside a subgraph is resolved, the child run resumes, but the parent run is never resumed. The parent run stays permanently in `WAITING_APPROVAL`.
**Why it happens:** The current `resolve_approval` API endpoint (approval_api.py lines 104-160) resumes only the run that created the approval. It has no concept of a parent run that is also waiting.
**How to avoid:** Two approaches (both work):
- **Option A (Recommended):** When the approval is resolved and the child run completes, the child run's completion triggers parent resumption. The `SubgraphExecutor` detects that the child is complete and the parent has `pending_subgraph` metadata, then continues the parent _drive() loop. This happens naturally if `resume_graph()` for the parent re-enters the `_drive()` loop which re-encounters the SubgraphNode, detects `pending_subgraph`, and resumes the child.
- **Option B:** Add a `parent_run_id` field to `ApprovalRecord` so the approval API can cascade resume to the parent. This requires modifying the approval service.
**Warning signs:** Parent runs stuck in `WAITING_APPROVAL` after child subgraph approvals are resolved.

### Pitfall 4: Node ID Namespace Collision with Parallel Execution
**What goes wrong:** A SubgraphNode has `parallel_config` set, causing fan-out on the subgraph's output. But the subgraph's namespaced node IDs don't account for branch indices.
**Why it happens:** Both parallel and subgraph features modify node IDs. Their prefixing strategies can clash.
**How to avoid:** The SubgraphNode should NOT be combinable with `parallel_config`. The fan-out feature operates on output lists from dispatch; SubgraphNode is handled before dispatch. If a user wants to run a subgraph in parallel, they should use a parent node that generates the list and a SubgraphNode per branch (via parallel fan-out of the parent, where each branch independently calls the subgraph). For Phase 39, validate and reject `SubgraphNode` with `parallel_config` set (similar to how Phase 38 rejects `HumanApprovalNode` with `parallel_config`).
**Warning signs:** `ParallelExecutor` attempting to split subgraph output, crashing on missing `split_path`.

### Pitfall 5: Graph Model Validation Failure on SubgraphNode
**What goes wrong:** Adding `SubgraphNode` to the discriminated union breaks existing graph deserialization because `SubgraphNode` requires a `subgraph` field that existing serialized graphs don't have.
**Why it happens:** Pydantic's discriminated union uses `node_type` to select the right model. Existing serialized graphs only have `node_type` values of "agent", "executable_unit", and "human_approval". They will NEVER match "subgraph" accidentally. The union is safe to extend.
**How to avoid:** No action needed. The discriminated union only selects `SubgraphNode` when `node_type == "subgraph"`. Existing graphs don't have this node type. Forward compatibility is guaranteed. Backward compatibility is maintained because `SubgraphNode` is a new variant, not a modification. [VERIFIED: Pydantic v2 discriminated union behavior]
**Warning signs:** None expected. Only worth testing explicitly with a round-trip serialize/deserialize of a graph containing a SubgraphNode.

### Pitfall 6: Subgraph Thread Sharing State Leak
**What goes wrong:** In "inherit" mode, agents in the subgraph share the parent's thread. If a subgraph agent writes to thread memory, the parent's agents see that data on subsequent nodes. This may be surprising.
**Why it happens:** Thread sharing is intentional (per D-08 "agents in subgraph see parent thread memory"), but the bidirectional nature may not be obvious.
**How to avoid:** This is working as designed. Document that "inherit" means FULL thread sharing, including writes. If users want read-only sharing, they need "isolated" mode with explicit data passing via input/output contracts. The SubgraphNodeData docstring should be clear about this.
**Warning signs:** Tests failing because thread memory state is different than expected after subgraph execution.

### Pitfall 7: DeploymentService Resolution Requires Database
**What goes wrong:** Tests that create a SubgraphNode but don't set up a database fail because SubgraphResolver calls DeploymentService which needs a deployment_repository backed by a database.
**Why it happens:** The SubgraphResolver depends on DeploymentService which depends on SQLiteDeploymentRepository which depends on AsyncDatabase.
**How to avoid:** The SubgraphExecutor should be optional (None) on RuntimeOrchestrator, just like `parallel_executor` has a default. When `subgraph_executor` is None and a SubgraphNode is encountered, raise a clear error: "SubgraphExecutor not configured -- cannot execute SubgraphNode". For tests, either mock the resolver or provide a test-scoped in-memory database (the existing test fixtures already do this).
**Warning signs:** Test failures in unrelated tests that don't use subgraphs.

## Code Examples

Verified patterns from the existing codebase:

### Existing Discriminated Union Pattern
```python
# Source: graph/models.py lines 244-247 [VERIFIED: codebase inspection]
Node = Annotated[
    AgentNode | ExecutableUnitNode | HumanApprovalNode,
    Field(discriminator="node_type"),
]
# Extension: Add SubgraphNode to the union
# Node = Annotated[
#     AgentNode | ExecutableUnitNode | HumanApprovalNode | SubgraphNode,
#     Field(discriminator="node_type"),
# ]
```

### Existing DeploymentService Resolution
```python
# Source: deployments/service.py lines 100-102 [VERIFIED: codebase inspection]
async def get(self, deployment_ref: str, version: int | None = None) -> Deployment | None:
    """Load the latest or a specific deployment version."""
    return await self.deployment_repository.get(deployment_ref, version)

# The Deployment model (deployments/models.py line 35) carries serialized_graph:
# serialized_graph: str  -- JSON string of the full Graph
```

### Existing Run Creation Pattern
```python
# Source: runtime.py lines 118-131 [VERIFIED: codebase inspection]
run = Run(
    graph_version_ref=self._graph_version_ref(graph),
    deployment_ref=deployment_ref or graph.graph_id,
    thread_id=thread_id or "",
    current_node_ids=[],
    pending_node_ids=[self._entry_step(graph)],
    metadata=self._initial_metadata(graph, initial_input),
)
persisted = await self.run_repository.create(run)
persisted.status = RunStatus.RUNNING
persisted.touch()
persisted = await self.run_repository.put(persisted)
```

### Existing HumanApprovalNode Handling (Pre-Dispatch Pattern)
```python
# Source: runtime.py lines 226-261 [VERIFIED: codebase inspection]
# SubgraphNode should follow this SAME pattern -- handled in _drive()
# BEFORE _dispatch_node(), not inside _dispatch_node().
if isinstance(node, HumanApprovalNode):
    # ... create approval, set WAITING_APPROVAL, checkpoint, return
```

### Existing ParallelExecutor Injection Pattern
```python
# Source: runtime.py line 100 [VERIFIED: codebase inspection]
# Phase 38: Parallel fan-out/fan-in executor.
parallel_executor: ParallelExecutor = ParallelExecutor()

# SubgraphExecutor follows the same pattern:
# subgraph_executor: SubgraphExecutor | None = None
# (None because it requires DeploymentService, unlike ParallelExecutor
# which is stateless. None triggers a clear error if a SubgraphNode
# is encountered without configuration.)
```

### Existing PolicyGuard Intersection (Governance Inheritance Primitive)
```python
# Source: policy/guard.py lines 117-136 [VERIFIED: codebase inspection]
def _allowed_capabilities(
    self,
    policies: list[PolicyDefinition],
) -> set[Capability] | None:
    allowed_lists = [
        set(policy.allowed_capabilities) for policy in policies if policy.allowed_capabilities
    ]
    if not allowed_lists:
        return None
    # Multiple allow-lists get stricter together
    allowed = allowed_lists[0]
    for policy_allowed in allowed_lists[1:]:
        allowed &= policy_allowed  # INTERSECTION -- parent ceiling enforced automatically
    return allowed
```

## Architecture Deep Dive: Approval Propagation

The hardest engineering challenge in this phase is approval propagation. Here is the full flow:

### Happy Path (No Approval)
```
Parent _drive() -> encounters SubgraphNode
  -> SubgraphExecutor creates child Run
  -> SubgraphExecutor calls _drive(subgraph, child_run)
  -> Child _drive() runs to completion
  -> Child Run.status = COMPLETED, final_output = {...}
  -> SubgraphExecutor returns child_run to parent
  -> Parent uses child_run.final_output as SubgraphNode's output
  -> Parent continues _drive() loop
```

### Approval Path
```
Parent _drive() -> encounters SubgraphNode
  -> SubgraphExecutor creates child Run
  -> SubgraphExecutor calls _drive(subgraph, child_run)
  -> Child _drive() encounters HumanApprovalNode
  -> Child Run.status = WAITING_APPROVAL
  -> _drive() returns child_run to SubgraphExecutor
  -> SubgraphExecutor detects WAITING_APPROVAL
  -> Parent run transitions to WAITING_APPROVAL
  -> Parent stores pending_subgraph metadata:
     { child_run_id, node_id, graph_ref }
  -> Parent re-queues SubgraphNode at front of pending_node_ids
  -> Parent _drive() returns run with WAITING_APPROVAL

  ... time passes, human resolves approval ...

Parent resume_graph(parent_run_id) called
  -> Parent _drive() re-enters
  -> Pops SubgraphNode (re-queued)
  -> Detects pending_subgraph metadata
  -> Calls resume_graph(subgraph, child_run_id)
  -> Child _drive() resumes after approval
  -> Child runs to completion (or hits another approval)
  -> Parent processes child result or re-pauses
```

### Key Implementation Detail: Resume Detection

When `_drive()` encounters a `SubgraphNode` and `run.metadata.get("pending_subgraph")` exists with a matching `node_id`, it means we are RESUMING after an approval pause. In this case:
1. Do NOT create a new child Run
2. Instead, call `self.resume_graph(subgraph, child_run_id)` to continue the existing child
3. The child _drive() handles the already-resolved approval via its existing `_consume_side_effect_approval` logic

This means the `SubgraphNode` handling in `_drive()` has TWO code paths:
- **First encounter:** Resolve subgraph, create child Run, call `_drive(subgraph, child_run)`
- **Resume after approval:** Resolve subgraph (again), call `resume_graph(subgraph, child_run_id)`

Both paths converge on the same outcome check: is the child run COMPLETED or still WAITING_APPROVAL?

### Nested Approval Propagation

If a subgraph itself contains a SubgraphNode with an approval, the propagation is naturally recursive:
```
Grandparent pauses -> Parent pauses -> Child pauses
Child approves -> Parent resumes child -> Parent completes -> Grandparent resumes parent -> Grandparent continues
```

Each level stores its own `pending_subgraph` metadata. The depth counter tracks nesting. The resume chain unwinds naturally through recursive `resume_graph` calls.

## Architecture Deep Dive: Node ID Namespacing

### Why Namespacing Is Needed

Without namespacing, a subgraph could have a node with the same ID as a parent node. This would cause:
- `_node_by_id()` in the parent to find the wrong node
- `node_visit_counts` to be incorrectly incremented
- Audit records to be attributed to the wrong node

### When to Namespace

Namespacing happens at the SubgraphExecutor level, AFTER resolving the graph and BEFORE calling `_drive()`. The namespaced graph is a copy (via `model_copy`) -- the original deployment's serialized graph is never modified.

### Namespace Format

`subgraph:{graph_ref}:{depth}:{original_node_id}`

Example: For a subgraph named "data-processor" at depth 1, a node "transform" becomes `subgraph:data-processor:1:transform`.

### Impact on Other Systems

- **Audit records:** Node IDs in audit records carry the prefix. This is intentional -- it makes the audit trail unambiguous about which nesting level a node executed at.
- **Execution history:** The child Run's execution history uses namespaced node IDs. When merged into parent metadata for traceability, the prefix distinguishes child entries from parent entries.
- **Conditions/edges:** Edge source_node_id and target_node_id are also namespaced, so condition evaluation and edge traversal work correctly within the subgraph scope.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Airflow SubDagOperator (graph flattening) | Airflow TaskGroup (composition without flattening) | Airflow 2.0 (2020) | SubDagOperator deprecated; proved that flattening is wrong |
| LangGraph subgraph (compile-time embedding) | LangGraph compiled subgraphs with shared state | LangGraph 0.2+ (2024) | Compile-time composition, not runtime resolution |
| Prefect flow-of-flows | Prefect subflow calls | Prefect 2.0 (2022) | Runtime composition with separate flow runs |

**Industry pattern:** All modern orchestrators (Prefect, Temporal, Airflow) have converged on subgraph-as-separate-execution-scope rather than flattening. The Zeroth approach (child Run, recursive _drive()) aligns with this consensus. [ASSUMED -- based on training data about Prefect/Temporal/Airflow architectures]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Approval propagation via pending_subgraph metadata and resume_graph chain is sufficient; no changes to ApprovalService needed | Approval Propagation Deep Dive | If the approval API needs to directly trigger parent resumption, the approval_api.py and ApprovalService require modification |
| A2 | SubgraphNode should NOT be combinable with parallel_config (reject at validation) | Pitfall 4 | If users need to fan-out into parallel subgraph invocations, the interaction between parallel and subgraph features needs design |
| A3 | Node ID namespacing is sufficient for collision prevention; no runtime scope object needed | Pattern 7 | If namespaced IDs cause issues with existing systems that parse node_id format (e.g., audit queries), a scope object may be needed |
| A4 | The SubgraphExecutor should be None-optional on RuntimeOrchestrator (not default-constructed) | Pattern 5 | If it should be default-constructed like ParallelExecutor, a no-op default that raises on use may be cleaner |
| A5 | The same RuntimeOrchestrator instance can safely call _drive() recursively because it is stateless between calls | Anti-Patterns | If the orchestrator accumulates state between _drive() calls (e.g., via mutable instance fields), recursive calls could interfere |
| A6 | Industry convergence on subgraph-as-separate-scope (Prefect, Temporal, Airflow) | State of the Art | Based on training data; specific version dates may be inaccurate |

## Open Questions (RESOLVED)

1. **SUBG requirement definitions** RESOLVED: Planner maps D-01..D-12 to SUBG-01..SUBG-08 per Phase Requirements table above.
   - What we know: The phase description references SUBG-01 through SUBG-08, and the CONTEXT.md has 12 detailed decisions
   - What's unclear: SUBG-01 through SUBG-08 are not defined in any REQUIREMENTS.md file yet
   - Recommendation: The planner should map CONTEXT.md decisions D-01 through D-12 to the 8 SUBG requirements. The mapping in the Phase Requirements section above is my best interpretation.

2. **SubgraphNode + parallel_config interaction** RESOLVED: Reject for Phase 39 — parallel subgraph invocation is a future enhancement.
   - What we know: Phase 38 rejects HumanApprovalNode with parallel_config
   - What's unclear: Should SubgraphNode with parallel_config be rejected, or should it fan out by running the subgraph N times (once per branch item)?
   - Recommendation: Reject for Phase 39 (A2 above). Parallel subgraph invocation is a future enhancement that requires careful design of child Run multiplexing.

3. **Contract compatibility validation at SubgraphNode edges** RESOLVED: Runtime validation only for Phase 39.
   - What we know: CONTEXT.md lists this as Claude's discretion
   - What's unclear: Whether the parent edge's output contract should be validated against the subgraph's entry node's input contract at graph authoring time or at runtime
   - Recommendation: Runtime validation only for Phase 39. The SubgraphExecutor validates that the input_payload structure matches what the subgraph's entry node expects. Compile-time validation requires resolving subgraph references during graph publishing, which adds complexity. Runtime validation with clear error messages is sufficient.

4. **Approval API awareness of parent runs** RESOLVED: Parent-resume-child pattern works transparently, no API changes needed.
   - What we know: The current approval resolution API resumes only the direct run_id on the approval record
   - What's unclear: Whether the approval API needs to know about parent_run_id chains, or if the parent-resume-child pattern handles this transparently
   - Recommendation: The parent-resume-child pattern should work transparently. When `resume_graph(parent_run_id)` is called, the parent's _drive() loop encounters the SubgraphNode with pending_subgraph metadata, resumes the child, and continues. No approval API changes needed. Test this flow end-to-end.

5. **Subgraph executor injection: None-optional vs default-constructed** RESOLVED: None-optional with clear error on SubgraphNode encounter.
   - What we know: ParallelExecutor is default-constructed (no dependencies). SubgraphExecutor needs DeploymentService.
   - What's unclear: Whether SubgraphExecutor should be None (requires explicit bootstrap wiring) or have a no-op default
   - Recommendation: None with clear error on SubgraphNode encounter. This matches the pattern used by `audit_repository`, `policy_guard`, `approval_service`, etc. The bootstrap wiring creates the executor when DeploymentService is available.

## Sources

### Primary (HIGH confidence)
- [VERIFIED: codebase inspection] `src/zeroth/core/orchestrator/runtime.py` -- _drive() loop (1315 lines), _dispatch_node(), run_graph(), resume_graph(), approval handling, parallel fan-out integration
- [VERIFIED: codebase inspection] `src/zeroth/core/graph/models.py` -- Node discriminated union, NodeBase fields, Graph model, SubgraphNode extension point
- [VERIFIED: codebase inspection] `src/zeroth/core/runs/models.py` -- Run model fields (confirmed parent_run_id ABSENT), RunStatus enum, thread_id handling
- [VERIFIED: codebase inspection] `src/zeroth/core/deployments/service.py` -- DeploymentService.get() API signature and behavior
- [VERIFIED: codebase inspection] `src/zeroth/core/deployments/models.py` -- Deployment.serialized_graph field
- [VERIFIED: codebase inspection] `src/zeroth/core/policy/guard.py` -- PolicyGuard.evaluate() and _allowed_capabilities() intersection semantics
- [VERIFIED: codebase inspection] `src/zeroth/core/audit/models.py` -- NodeAuditRecord fields and execution_metadata extensibility
- [VERIFIED: codebase inspection] `src/zeroth/core/parallel/` -- Phase 38 package structure, models, executor, errors as reference pattern
- [VERIFIED: codebase inspection] `src/zeroth/core/service/bootstrap.py` -- Service injection patterns, ServiceBootstrap dataclass fields
- [VERIFIED: codebase inspection] `src/zeroth/core/service/approval_api.py` -- Approval resolution flow and resume mechanism
- [VERIFIED: codebase inspection] GovernAI RunState source -- confirmed no parent_run_id field on base class
- [VERIFIED: runtime] Python 3.12.12, Pydantic 2.12.5, pytest-asyncio 1.3.0

### Secondary (MEDIUM confidence)
- [CITED: Phase 38 RESEARCH.md] _drive() loop architecture, BranchContext isolation patterns, asyncio.gather behavior
- [CITED: Phase 38 VERIFICATION.md] Verification that Phase 38 parallel execution is fully implemented and passing

### Tertiary (LOW confidence)
- [ASSUMED] Airflow SubDagOperator deprecation timeline and reasoning
- [ASSUMED] LangGraph and Prefect subgraph architecture comparison
- [ASSUMED] Industry convergence on subgraph-as-separate-scope pattern

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies, all stdlib + existing pydantic [VERIFIED]
- Architecture: HIGH -- all integration points verified via codebase inspection; _drive() loop, DeploymentService, PolicyGuard, Run model fully mapped
- Pitfalls: HIGH -- parent_run_id absence confirmed (critical finding); approval propagation flow traced through approval_api.py; node ID collision risk analyzed
- Governance inheritance: HIGH -- PolicyGuard intersection semantics verified in source code; parent-ceiling model confirmed as automatic consequence of policy list prepending

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable domain -- orchestrator code changes only within this phase and Phase 38)
