---
phase: 39-subgraph-composition
verified: 2026-04-13T03:11:34Z
status: human_needed
score: 5/5
overrides_applied: 0
human_verification:
  - test: "Run a full subgraph composition end-to-end with a real DeploymentService and SQLite DB, deploying a child graph, referencing it from a parent graph as a SubgraphNode, and running the parent graph to completion"
    expected: "Parent run completes with child run's output, parent run's execution_history has subgraph audit records, child run has parent_run_id set"
    why_human: "Integration tests use mocked orchestrator internals -- a full bootstrap-to-completion run with real SubgraphResolver and repository would catch any wiring issue that mocks hide"
  - test: "Deploy a parent graph with a SubgraphNode pointing to a child graph that contains a HumanApprovalNode, run the parent, verify it pauses, then resolve the approval and resume the parent"
    expected: "Parent transitions to WAITING_APPROVAL, pending_subgraph metadata stored, resume cascades to child, child completes, parent completes with child output"
    why_human: "Approval propagation involves multiple async resume cycles and real Run persistence -- mock-based tests may not catch ordering or state issues"
  - test: "Verify visual audit trail in a UI or log output showing parent run -> subgraph child run linkage with namespaced node IDs"
    expected: "Audit records clearly show parent_run_id, subgraph_run_id, and node IDs with 'subgraph:{ref}:{depth}:' prefix"
    why_human: "Audit trail readability and traceability is a human-judgment quality concern"
---

# Phase 39: Subgraph Composition Verification Report

**Phase Goal:** A graph can reference another published graph as a nested subgraph node, with the orchestrator entering the subgraph as a scoped execution that inherits governance, shares thread memory, and propagates approvals back to the parent
**Verified:** 2026-04-13T03:11:34Z
**Status:** human_needed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A graph author can add a subgraph node that references another published graph by name; the subgraph's entry contract must be compatible with the referencing edge's mapping output, and the subgraph's final output maps back to the parent graph's expected input | VERIFIED | SubgraphNode exists in Node union (graph/models.py:275-277), SubgraphNodeData accepts graph_ref/version/thread_participation/max_depth (subgraph/models.py:14-33), round-trip serialization confirmed via behavioral spot-check, executor passes input_payload to child and returns final_output to parent (executor.py:133, runtime.py:384-386). Contract compatibility relies on existing input_contract_ref/output_contract_ref on NodeBase (inherited by SubgraphNode). |
| 2 | The orchestrator enters the subgraph as a nested scope sharing the parent's thread_id (configurable); agents inside subgraphs participate in the same thread memory; the parent's policies apply as a baseline that the subgraph can further restrict but not relax | VERIFIED | SubgraphExecutor.execute() sets child thread_id from parent when thread_participation="inherit", empty when "isolated" (executor.py:119-123). merge_governance() prepends parent policy_bindings to subgraph (resolver.py:113-135). Tests confirm both thread modes and governance merge (test_executor.py, test_resolver.py, test_integration.py). |
| 3 | If a HumanApprovalNode inside a subgraph pauses execution, the parent run transitions to WAITING_APPROVAL; resolution resumes the subgraph and eventually the parent run | VERIFIED | runtime.py:368-381 (Path A: child WAITING_APPROVAL -> parent WAITING_APPROVAL with pending_subgraph metadata, re-queue node_id). runtime.py:283-347 (Path B: resume detects pending_subgraph, re-resolves, calls resume_graph on child, clears metadata on completion). 12 approval propagation tests + 2 integration approval tests all pass. |
| 4 | The same subgraph can be referenced by multiple parent graphs and at multiple points within a single parent; subgraph references can pin to a specific deployment version or float to the latest active deployment; nested subgraphs (subgraph within a subgraph) are supported with a configurable depth limit | VERIFIED | SubgraphNodeData.version: int|None=None allows version pin or latest (models.py:27). Resolver passes version to DeploymentService.get() (resolver.py:50). Multi-reference tested (test_integration.py: two SubgraphNode instances produce distinct child runs). Depth limit enforced via max_depth field (ge=1, le=10) and checked in executor (executor.py:96-102). Cycle detection via visited_subgraph_refs (executor.py:105-107). |
| 5 | Audit records from subgraph execution link to the parent run via parent_run_id, and node IDs are namespaced to prevent collisions across nesting levels | VERIFIED | child Run created with parent_run_id=parent_run.run_id (executor.py:148). namespace_subgraph() prefixes all node_ids, edge IDs, entry_step with "subgraph:{graph_ref}:{depth}:" (resolver.py:65-110). Audit records include subgraph_run_id, subgraph_graph_ref, subgraph_depth (runtime.py:388-393). Behavioral spot-check confirmed namespace prefix format. |

**Score:** 5/5 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/zeroth/core/subgraph/__init__.py` | Public re-exports for subgraph package | VERIFIED | Exports models, errors, lazy-loads SubgraphExecutor |
| `src/zeroth/core/subgraph/models.py` | SubgraphNodeData Pydantic model | VERIFIED | class SubgraphNodeData with graph_ref, version, thread_participation, max_depth |
| `src/zeroth/core/subgraph/errors.py` | Error hierarchy (5 classes) | VERIFIED | SubgraphError, SubgraphDepthLimitError, SubgraphResolutionError, SubgraphExecutionError, SubgraphCycleError |
| `src/zeroth/core/subgraph/resolver.py` | SubgraphResolver, namespace_subgraph, merge_governance | VERIFIED | All 3 present and substantive (136 lines) |
| `src/zeroth/core/subgraph/executor.py` | SubgraphExecutor with execute() | VERIFIED | 165 lines, full depth/cycle/resolve/namespace/governance/create/drive flow |
| `src/zeroth/core/graph/models.py` | SubgraphNode in Node union | VERIFIED | SubgraphNode(NodeBase) at line 245, Node union updated at line 275 |
| `src/zeroth/core/runs/models.py` | Run.parent_run_id field | VERIFIED | parent_run_id: str | None = None at line 127 |
| `src/zeroth/core/orchestrator/runtime.py` | SubgraphNode detection in _drive(), subgraph_executor field, approval propagation | VERIFIED | subgraph_executor field at line 112, SubgraphNode isinstance check at line 274, Path A (first encounter) and Path B (resume) fully implemented |
| `src/zeroth/core/service/bootstrap.py` | SubgraphExecutor wiring | VERIFIED | SubgraphResolver and SubgraphExecutor created and wired at lines 374-380, passed to ServiceBootstrap at line 470 |
| `tests/subgraph/test_models.py` | Model and type tests | VERIFIED | 25 tests |
| `tests/subgraph/test_resolver.py` | Resolver, namespace, governance tests | VERIFIED | 14 tests |
| `tests/subgraph/test_executor.py` | Executor unit tests | VERIFIED | 13 tests |
| `tests/subgraph/test_drive_subgraph.py` | Drive loop integration tests | VERIFIED | 12 tests |
| `tests/subgraph/test_approval_propagation.py` | Approval propagation tests | VERIFIED | 12 tests |
| `tests/subgraph/test_integration.py` | Comprehensive integration tests | VERIFIED | 14 tests |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| subgraph/models.py | graph/models.py | SubgraphNode extends NodeBase, added to Node union | WIRED | `class SubgraphNode(NodeBase)` at graph/models.py:245, Node union at :275 |
| subgraph/resolver.py | deployments/service.py | SubgraphResolver.resolve() calls DeploymentService.get() | WIRED | `deployment_service.get(graph_ref, version)` at resolver.py:50 |
| subgraph/executor.py | orchestrator/runtime.py | SubgraphExecutor.execute() calls orchestrator._drive() | WIRED | `orchestrator._drive(merged, child_run)` at executor.py:160 |
| orchestrator/runtime.py | subgraph/executor.py | _drive() delegates SubgraphNode to subgraph_executor.execute() | WIRED | `self.subgraph_executor.execute(...)` at runtime.py:351 |
| service/bootstrap.py | subgraph/executor.py | bootstrap wires SubgraphExecutor with DeploymentService | WIRED | SubgraphResolver + SubgraphExecutor created and assigned at bootstrap.py:374-380 |
| orchestrator/runtime.py | orchestrator/runtime.py | resume_graph -> _drive() -> pending_subgraph -> resume_graph(child) | WIRED | Path B at runtime.py:287-347 re-resolves, calls self.resume_graph(subgraph, child_run_id) |
| graph/__init__.py | graph/models.py | SubgraphNode exported | WIRED | Import and __all__ entry at graph/__init__.py:21,41-42 |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| executor.py | input_payload | Passed from orchestrator _drive() loop | Yes -- flows through from edge mappings | FLOWING |
| executor.py | child_run.final_output | Set by child _drive() completion | Yes -- child run produces output from agent execution | FLOWING |
| runtime.py (SubgraphNode handler) | output_data | child_run.final_output | Yes -- used as node output for history and next-node planning | FLOWING |
| runtime.py (Path B resume) | output_data | child_run.final_output after resume | Yes -- same flow as Path A | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Module imports | `python -c "from zeroth.core.subgraph import ..."` | All 7 imports succeed | PASS |
| SubgraphNode serialization round-trip | `serialize_graph + deserialize_graph` | SubgraphNode preserved with all fields | PASS |
| Node ID namespacing | `namespace_subgraph(g, 'child-graph', 1)` | node_id="subgraph:child-graph:1:sub1", entry_step prefixed, original unchanged | PASS |
| Governance merge | `merge_governance(parent, child)` | policy_bindings=['deny-all', 'allow-read'], original child unchanged | PASS |
| Run parent_run_id | `Run(..., parent_run_id='parent-123')` | parent_run_id='parent-123' | PASS |
| All subgraph tests | `uv run pytest tests/subgraph/ -v` | 90 passed in 0.09s | PASS |
| Orchestrator regression | `uv run pytest tests/orchestrator/ -v` | 11 passed in 1.15s (zero regressions) | PASS |
| Lint check | `uv run ruff check` all modified files | All checks passed | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| SUBG-01 | 39-01 | SubgraphNode with graph_ref, version, thread_participation, max_depth | SATISFIED | SubgraphNodeData model, SubgraphNode in Node union, discriminated union serialization |
| SUBG-02 | 39-02 | Orchestrator resolves subgraph at runtime, executes via recursive _drive() | SATISFIED | SubgraphExecutor.execute() resolves via resolver, calls orchestrator._drive() |
| SUBG-03 | 39-01 | Child Run linked to parent via parent_run_id | SATISFIED | Run.parent_run_id field, executor sets parent_run_id=parent_run.run_id |
| SUBG-04 | 39-02 | Parent governance acts as ceiling -- subgraph can restrict not relax | SATISFIED | merge_governance() prepends parent policy_bindings; PolicyGuard intersection semantics |
| SUBG-05 | 39-02 | Thread participation configurable: inherit shares thread_id, isolated creates new | SATISFIED | executor.py:119-123 branch on thread_participation, tests confirm both modes |
| SUBG-06 | 39-03 | Approval pauses propagate to parent, resolution cascades back | SATISFIED | Path A and Path B in runtime.py, 12 approval propagation tests, 2 integration approval tests |
| SUBG-07 | 39-01 | Depth tracking with SubgraphDepthLimitError | SATISFIED | max_depth field (ge=1, le=10), depth check in executor.py:96-102, tested |
| SUBG-08 | 39-01, 39-03 | Node IDs namespaced with subgraph:{ref}:{depth}: prefix | SATISFIED | namespace_subgraph() in resolver.py:65-110, tested in models, resolver, and integration tests |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none found) | - | - | - | All source files clean: no TODOs, no placeholders, no empty returns, no hardcoded empty data |

### Human Verification Required

### 1. Full Bootstrap-to-Completion Subgraph Run

**Test:** Deploy a child graph (single AgentNode), create a parent graph with a SubgraphNode referencing the child, bootstrap the full service, and run the parent graph end-to-end using `run_graph()`.
**Expected:** Parent run completes with child's output. Child run exists in the repository with parent_run_id set. Parent's execution_history contains subgraph audit records with subgraph_run_id.
**Why human:** Integration tests mock orchestrator internals (run_repository, agent runners). A real bootstrap run with SQLite and mocked LLM provider would validate the full wiring path including SubgraphExecutor injection via bootstrap.

### 2. Approval Propagation Round-Trip

**Test:** Deploy a child graph containing a HumanApprovalNode. Create a parent graph with a SubgraphNode referencing it. Run the parent. Verify WAITING_APPROVAL state on both runs. Resolve the approval. Resume the parent. Verify completion.
**Expected:** Both parent and child run transition through WAITING_APPROVAL. Resume cascades correctly. Parent completes with child's post-approval output.
**Why human:** The multi-step pause/resume cycle involves real async state transitions and Run persistence that mock-based tests may not fully exercise.

### 3. Audit Trail Readability

**Test:** After running a subgraph composition, inspect the execution_history of both parent and child runs for clarity of traceability.
**Expected:** Node IDs carry the "subgraph:{ref}:{depth}:" prefix. Audit records include subgraph_run_id. The parent-to-child linkage is unambiguous for a human auditor.
**Why human:** Audit trail quality is a human-judgment concern -- automated checks confirm fields exist but not that the trail is usable for production debugging.

### Gaps Summary

No gaps found. All 5 roadmap success criteria are verified with strong evidence. All 8 SUBG requirements are satisfied. All 90 tests pass with zero regressions across orchestrator tests. All behavioral spot-checks pass. Lint is clean.

Three items require human verification: (1) a full bootstrap-to-completion run to validate real wiring beyond mocks, (2) an approval propagation round-trip with real persistence, and (3) audit trail readability assessment.

---

_Verified: 2026-04-13T03:11:34Z_
_Verifier: Claude (gsd-verifier)_
