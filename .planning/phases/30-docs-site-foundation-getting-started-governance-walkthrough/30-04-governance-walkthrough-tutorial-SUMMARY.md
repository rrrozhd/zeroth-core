---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 04
subsystem: docs
tags: [docs, tutorial, governance, approval, policy, audit]
requires:
  - zeroth.core.examples.quickstart
  - zeroth.core.policy.guard
  - zeroth.core.policy.registry
  - zeroth.core.service.audit_api
  - zeroth.core.service.approval_api
  - zeroth.core.service.bootstrap
provides:
  - examples/governance_walkthrough.py (single script exercising all three governance primitives)
  - docs/tutorials/governance-walkthrough.md (Phase 30 marquee tutorial, DOCS-05)
  - Four new shape tests in tests/test_docs_phase30.py
affects:
  - tests/test_docs_phase30.py
tech-stack:
  added: []
  patterns:
    - Bootstrap two in-process services against one shared SQLite DB — one for the approval-gated graph and one for the policy-blocked graph — so the example can demonstrate both scenarios without cross-deployment contamination of the orchestrator runtime state
    - Register each Capability enum value under its own ref (e.g. "network_write") in a freshly-constructed CapabilityRegistry, because build_demo_graph_with_policy stores capability_bindings as raw enum values rather than a bespoke ref scheme
    - Drive runs via orchestrator.run_graph(...) directly (not POST /runs) because bootstrap_service with enable_durable_worker=False has no dispatch path that would drive a persisted PENDING run synchronously, while still exercising the real HTTP surface for /approvals/.../resolve, /runs/{id}/timeline, and /deployments/{ref}/audits
key-files:
  created:
    - examples/governance_walkthrough.py
  modified:
    - docs/tutorials/governance-walkthrough.md
    - tests/test_docs_phase30.py
key-decisions:
  - Drive runs via `orchestrator.run_graph()` directly instead of POST /runs because the Run API persists the run as PENDING and relies on a durable worker to pick it up; with `enable_durable_worker=False` there is no in-request dispatch path. The HTTP surface is still exercised for the three governance-specific endpoints the tutorial is teaching (approvals resolve, run timeline, deployment audits), which are the endpoints readers would actually call.
  - Use `Capability.NETWORK_WRITE` via `build_demo_graph_with_policy([Capability.NETWORK_WRITE])` without touching the quickstart helper — verified end-to-end with a direct `PolicyGuard.evaluate` call showing `decision=deny, reason="capability denied: network_write"`. No changes were needed to `zeroth.core.examples.quickstart`.
  - Register every `Capability` enum value in the `CapabilityRegistry` under its own value-as-ref (`cr.register(cap.value, cap)` for each) rather than hand-picking just `network_write`, so the example is robust if the plan adds more denied capabilities later.
  - Assign `ServiceRole.AUDITOR` in addition to `OPERATOR` + `REVIEWER` to the demo API key so the `/runs/{id}/timeline` and `/deployments/{ref}/audits` calls (which require `Permission.AUDIT_READ`) go through on a single header.
requirements-completed: [DOCS-05]
duration: 10 min
completed: 2026-04-11
---

# Phase 30 Plan 04: Governance Walkthrough Tutorial Summary

Shipped Phase 30's marquee deliverable (**DOCS-05**): a single runnable script, `examples/governance_walkthrough.py`, that exercises all three Zeroth governance primitives — **approval gate**, **auditor timeline**, and **policy block** — end-to-end against one in-process bootstrap, plus a fully-written `docs/tutorials/governance-walkthrough.md` tutorial page that embeds the script via `pymdownx.snippets` and frames each scenario as the Zeroth differentiator vs LangGraph/CrewAI/AutoGen. Four new shape tests added to `tests/test_docs_phase30.py` assert page shape, example embed, three-scenario coverage, and SKIP-clean subprocess execution.

## What Shipped

### `examples/governance_walkthrough.py` (~320 LOC)

A single runnable script that:

1. SKIPs cleanly on missing `OPENAI_API_KEY` (exit 0 + stderr notice) — same pattern as `first_graph.py` and `approval_demo.py`.
2. Boots a tempdir SQLite database with Alembic migrations and registers a trivial `DemoPayload` contract under `contract://demo-input` / `contract://demo-output`.
3. **Scenario 1 — Approval gate.** Persists `build_demo_graph(include_approval=True)` as deployment `demo-governance-approval`, bootstraps a service with `enable_durable_worker=False`, replaces `orchestrator.agent_runners` with a litellm-backed `_LiteLLMAgentRunner`, mounts the FastAPI app on `httpx.ASGITransport`, then drives the run via `orchestrator.run_graph(...)`. The run pauses in `WAITING_APPROVAL` at the `HumanApprovalNode`. The example lists pending approvals via `approval_service.list_pending(...)`, takes the first `approval_id`, and POSTs to `/deployments/{ref}/approvals/{id}/resolve` with `{"decision": "approve"}` — the real HTTP endpoint. The response body contains the terminal run.
4. **Scenario 2 — Auditor.** Fetches `GET /runs/{run_id}/timeline` via the in-process `httpx.AsyncClient` and prints each `NodeAuditRecord`'s `node_id`, `status`, and any enforcement metadata. Uses the same `X-API-Key` header that scenario 1 used — the demo key holds `OPERATOR + REVIEWER + AUDITOR` roles so `Permission.AUDIT_READ` is granted.
5. **Scenario 3 — Policy block.** Persists `build_demo_graph_with_policy(denied_capabilities=[Capability.NETWORK_WRITE])` as deployment `demo-governance-blocked`, bootstraps a *second* service against that deployment, then **wires a `PolicyGuard`** onto `blocked_service.orchestrator.policy_guard` post-bootstrap with:
   - A `PolicyRegistry` containing `PolicyDefinition(policy_id="block-demo-caps", denied_capabilities=[Capability.NETWORK_WRITE])`.
   - A `CapabilityRegistry` where every `Capability` enum value is registered under its own value-as-ref (`cr.register(cap.value, cap)` in a loop).
   Drives a run via `orchestrator.run_graph(...)`. The orchestrator's `_enforce_policy` path evaluates the guard, gets `PolicyDecision.DENY` with reason `capability denied: network_write`, writes a rejected `NodeAuditRecord` with `execution_metadata.enforcement`, and calls `_fail_run(run, "policy_violation", ...)`. The internal status becomes `RunStatus.FAILED`; the public Run API surfaces this as `RunPublicStatus.TERMINATED_BY_POLICY`. The example then fetches `GET /deployments/{ref}/audits?run_id=...`, filters for records whose `execution_metadata.enforcement.decision == "deny"`, and prints them.

All three scenarios share one tempdir SQLite database and one demo API key. Both services are created via `bootstrap_service` → `create_app(service)` → `app.state.bootstrap = service` → `httpx.ASGITransport(app=app)`, matching the `examples/approval_demo.py` pattern.

### `docs/tutorials/governance-walkthrough.md` (~120 lines of prose + 1 full-file snippet embed)

Structure:

- **H1 Governance Walkthrough** + one-sentence framing ("approval gate, auditor, policy — one run, one graph, one bootstrap").
- **Why this matters** — positions Zeroth against LangGraph / CrewAI / AutoGen as the framework that ships the *governance* layer, not just the agents.
- **Prerequisites** — points back to Getting Started Install + First graph.
- **Running the walkthrough** — one-line `uv run python examples/governance_walkthrough.py` + admonition about `OPENAI_API_KEY`.
- **Scenario 1 — Approval gate** — explains `HumanApprovalNode`, `WAITING_APPROVAL`, and the `/approvals/.../resolve` HTTP endpoint.
- **Scenario 2 — Auditor** — explains `GET /runs/{id}/timeline`, the per-node structured `NodeAuditRecord` shape, and what "audit trail" means in Zeroth.
- **Scenario 3 — Policy block** — explains `PolicyDefinition`, `denied_capabilities`, `policy_bindings` on nodes, `PolicyGuard.evaluate`, and the `RunStatus.FAILED` (reason `policy_violation`) → `RunPublicStatus.TERMINATED_BY_POLICY` mapping.
- **Full example** — `--8<-- "governance_walkthrough.py"` embed of the whole file.
- **Where to next** — pointers to the source modules for approvals/audit/policy until the Phase 31 concept pages land.

### Shape tests (`tests/test_docs_phase30.py`)

Four new Plan 30-04 tests appended after the Plan 30-03 block (17 total tests in the file):

1. `test_governance_walkthrough_page_shape` — asserts file exists, starts with `# Governance Walkthrough`, and contains all three keywords (`approval`, `audit`, `policy`) case-insensitively.
2. `test_governance_walkthrough_embeds_example` — asserts `--8<--` and `governance_walkthrough.py` both appear in the page.
3. `test_governance_walkthrough_example_covers_three_scenarios` — asserts the example file references `approval`, `timeline`, `Capability`, and `NETWORK_WRITE`.
4. `test_governance_walkthrough_example_skips_cleanly` — spawns the example as a subprocess without `OPENAI_API_KEY` in the environment and asserts `returncode == 0` and `"SKIP" in stderr`.

## Tasks & Commits

| Task | Name                                                          | Commit    | Files                                                                   |
| ---- | ------------------------------------------------------------- | --------- | ----------------------------------------------------------------------- |
| 1    | Write examples/governance_walkthrough.py                      | `535ec14` | `examples/governance_walkthrough.py`                                    |
| 2    | Write governance tutorial page + Plan 30-04 shape tests       | `47717e1` | `docs/tutorials/governance-walkthrough.md`, `tests/test_docs_phase30.py`|

## Verification Results

- `OPENAI_API_KEY= uv run python examples/governance_walkthrough.py` → `SKIP: ...`, **exit 0**.
- `uv run ruff check examples/governance_walkthrough.py tests/test_docs_phase30.py` → **All checks passed**.
- `uv run ruff format examples/governance_walkthrough.py tests/test_docs_phase30.py` → reformatted on first write; clean on re-run.
- `uv run pytest tests/test_docs_phase30.py -v` → **17 passed** (1 scaffold + 7 plan-02 + 5 plan-03 + 4 plan-04).
- `uv run mkdocs build --strict` → **exit 0** (the stderr "MkDocs 2.0" notice is an upstream informational banner, not a strict failure).
- `PolicyGuard` denial path smoke-verified directly:
  ```
  decision: deny reason: capability denied: network_write
  ```
  via a one-shot `PolicyGuard.evaluate(...)` call against `build_demo_graph_with_policy([Capability.NETWORK_WRITE])`.

The real-LLM happy path was **not** exercised locally because the session has no `OPENAI_API_KEY` — per the Phase 30 pattern established by plans 02 and 03, that path runs in `.github/workflows/examples.yml` on pushes to `main` via the repo secret. The guarded slot for `examples/governance_walkthrough.py` in that workflow was already shipped by plan 03, so this plan required zero changes to the CI workflow.

If an `OPENAI_API_KEY` had been available, the verified-happy-path command would have been:

```bash
OPENAI_API_KEY=<redacted> uv run python examples/governance_walkthrough.py
```

Expected stdout: three scenario headers, one pending approval id, a timeline print of three-to-five node audit records, a blocked run with `status.value=failed` and `failure_state.reason=policy_violation`, one or more `DENIED at [tool]` audit lines, and a final `All three governance scenarios passed.`

## Capability / Node Combination That Triggers TERMINATED_BY_POLICY

Per the plan's output section request: the current runtime enforces `TERMINATED_BY_POLICY` (as `RunStatus.FAILED` with `failure_state.reason == "policy_violation"`, mapped to `RunPublicStatus.TERMINATED_BY_POLICY` by `run_api._failed_status`) via the following exact path:

1. `RuntimeOrchestrator._drive` → `_enforce_policy(graph, run, node, input_payload)` before each node runs.
2. `_enforce_policy` calls `self.policy_guard.evaluate(graph, node, run, input_payload)` when `self.policy_guard is not None`.
3. `PolicyGuard.evaluate` collects policies from `graph.policy_bindings + node.policy_bindings` via `self.policy_registry.resolve(ref)`, resolves `node.capability_bindings` via `self.capability_registry.resolve(ref)` into a `set[Capability]` of required capabilities, and unions all `policy.denied_capabilities` from the resolved policies into the denied set. If any required capability is in the denied set, returns `EnforcementResult(decision=DENY, reason=f"capability denied: {...}")`.
4. `_enforce_policy` writes a rejected `NodeAuditRecord` (status `"rejected"`, `execution_metadata.enforcement` containing the full `EnforcementResult.model_dump(mode="json")`) and calls `_fail_run(run, "policy_violation", result.reason)`.

The example triggers this path with:

- **Node:** the tool node produced by `build_demo_graph_with_policy(denied_capabilities=[Capability.NETWORK_WRITE])`, which has `policy_bindings=["block-demo-caps"]` and `capability_bindings=["network_write"]`.
- **Registered PolicyDefinition:** `PolicyDefinition(policy_id="block-demo-caps", denied_capabilities=[Capability.NETWORK_WRITE])`.
- **CapabilityRegistry:** every `Capability` enum value registered under its own value-as-ref (`network_write` → `Capability.NETWORK_WRITE`, etc.).

Because `bootstrap_service` does **not** wire a `PolicyGuard` by default, the example assigns it directly post-bootstrap: `blocked_service.orchestrator.policy_guard = PolicyGuard(policy_registry=..., capability_registry=...)`. This is the minimal seam the example needed and does not require any upstream runtime changes.

## Changes to `zeroth.core.examples.quickstart`

**None.** The plan offered the executor a choice between (a) using an already-enforced capability/node combo and (b) extending `build_demo_graph_with_policy` to produce a node shape the runtime would deny. Option (a) worked out of the box: the quickstart helper already sets the tool node's `capability_bindings` to the raw `Capability.value` strings, and those are exactly the refs the example registers in `CapabilityRegistry`. No upstream changes were needed.

## Success Criteria

- [x] DOCS-05 shipped: one tutorial page + one runnable example exercising approval gate + auditor + policy block
- [x] Example uses real `PolicyDefinition` / `Capability.NETWORK_WRITE` / `/runs/{id}/timeline` / `/deployments/{ref}/audits` / `/approvals/{id}/resolve` — no fakes, no mocks
- [x] Page has all five required sections (Why this matters, Prerequisites, Running, three Scenarios, Full example, Where to next) + full-file snippet embed
- [x] Script exit code 0 on SKIP path; the guarded slot in `.github/workflows/examples.yml` (shipped in plan 30-03) picks the new file up automatically
- [x] `uv run mkdocs build --strict` → green
- [x] `uv run pytest tests/test_docs_phase30.py -v` → 17 passed (4 new Plan 30-04 tests)
- [x] Ruff clean on `examples/governance_walkthrough.py` and `tests/test_docs_phase30.py`

## Deviations from Plan

**None** — plan executed exactly as written. Ruff's auto-formatter reflowed one multi-line f-string in the example script on first save (collapsing a parenthesized `resolve_url = (f"/deployments/...")` onto a single line); this is cosmetic, counted as formatter output rather than a deviation.

One note on the plan's guidance to "handle the reality gap honestly": the runtime *does* enforce `Capability.NETWORK_WRITE` denial for a tool node with `capability_bindings=["network_write"]` **provided** the runtime has a `PolicyGuard` wired with a `CapabilityRegistry` that maps the `"network_write"` ref to `Capability.NETWORK_WRITE`. `bootstrap_service` does not wire one by default, so the example assigns `orchestrator.policy_guard` directly post-bootstrap — a documented, tested seam (`@dataclass(slots=True) class RuntimeOrchestrator: policy_guard: PolicyGuard | None = None`). This is option (a) from the plan's guidance and required no upstream code changes.

## Threat Flags

None — the tutorial example and page do not introduce any new network endpoints, auth paths, or trust boundaries beyond what plans 30-01 through 30-03 already cover. The example uses the same in-process `httpx.ASGITransport` pattern and the same `demo-operator-key` constant as `approval_demo.py`; both stay inside the script's process and never leave via the network.

## Self-Check: PASSED

- [x] `examples/governance_walkthrough.py` exists on disk
- [x] `docs/tutorials/governance-walkthrough.md` exists on disk (rewritten from plan-02 placeholder)
- [x] `tests/test_docs_phase30.py` modified (4 new Plan 30-04 tests, 17 total)
- [x] Commit `535ec14` present in git log (`feat(30-04): add governance_walkthrough.py end-to-end example`)
- [x] Commit `47717e1` present in git log (`docs(30-04): fill Governance Walkthrough tutorial + shape tests`)
- [x] `uv run pytest tests/test_docs_phase30.py -v` → 17 passed
- [x] `uv run mkdocs build --strict` → exit 0
- [x] `OPENAI_API_KEY= uv run python examples/governance_walkthrough.py` → exit 0 (SKIP)
