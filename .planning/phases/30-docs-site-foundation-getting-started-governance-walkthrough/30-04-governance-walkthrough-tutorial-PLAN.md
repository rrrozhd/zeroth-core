---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 04
type: execute
wave: 2
depends_on: [01, 02]
files_modified:
  - examples/governance_walkthrough.py
  - docs/tutorials/governance-walkthrough.md
  - tests/test_docs_phase30.py
autonomous: true
requirements:
  - DOCS-05
tags: [docs, tutorial, governance, approval, policy, audit]
must_haves:
  truths:
    - "`python examples/governance_walkthrough.py` runs three scenarios end-to-end against a single in-process service: approval gate → auditor review → policy block — with a clean SKIP path when OPENAI_API_KEY is unset"
    - "Scenario 1 (approval gate) submits a run that pauses at a HumanApprovalNode, resolves via POST /deployments/{ref}/approvals/{id}/resolve, and shows the run transition to succeeded"
    - "Scenario 2 (auditor) fetches GET /runs/{id}/timeline and prints each NodeAuditRecord's node_id, status, and any policy decisions"
    - "Scenario 3 (policy block) deploys a variant graph whose tool node has `policy_bindings` pointing at a PolicyDefinition with `denied_capabilities=[Capability.NETWORK_WRITE]`, submits a run that triggers the denied capability, observes the run terminating with RunStatus.TERMINATED_BY_POLICY, and then fetches GET /deployments/{ref}/audits to surface the denial record"
    - "`docs/tutorials/governance-walkthrough.md` embeds `examples/governance_walkthrough.py` and explains each scenario in prose, framed as the Zeroth differentiator vs LangGraph/CrewAI"
    - "The example covers all three scenarios in a single script (not three separate scripts) per CONTEXT.md D-11"
  artifacts:
    - path: "examples/governance_walkthrough.py"
      provides: "Single script that exercises approval gate, auditor review, and policy block against one in-process service bootstrap"
      min_lines: 120
    - path: "docs/tutorials/governance-walkthrough.md"
      provides: "Long-form tutorial: overview, three scenario sections, each with an embedded snippet range + expected output, plus a 'why this matters' section"
      min_lines: 80
  key_links:
    - from: "examples/governance_walkthrough.py"
      to: "src/zeroth/core/policy/models.py"
      via: "PolicyDefinition(denied_capabilities=[Capability.NETWORK_WRITE])"
      pattern: "PolicyDefinition|Capability\\.NETWORK_WRITE"
    - from: "examples/governance_walkthrough.py"
      to: "src/zeroth/core/service/audit_api.py"
      via: "GET /runs/{run_id}/timeline + GET /deployments/{ref}/audits"
      pattern: "/timeline|/audits"
    - from: "docs/tutorials/governance-walkthrough.md"
      to: "examples/governance_walkthrough.py"
      via: "pymdownx.snippets embed (whole file or ranges)"
      pattern: "--8<--.*governance_walkthrough"
---

<objective>
Ship the Governance Walkthrough tutorial — a single runnable example
that demonstrates the three Zeroth differentiators (approval gate,
auditor, policy block) in one coherent flow, plus the prose tutorial
page that frames each scenario for the reader.

Purpose: Satisfy DOCS-05. This is the phase's marquee deliverable — it's
the page that distinguishes Zeroth from LangGraph, CrewAI, AutoGen,
etc., so it must be substantive, not toy.

Output: One runnable `examples/governance_walkthrough.py` (~120-180 LOC)
and one fully-written `docs/tutorials/governance-walkthrough.md`.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-CONTEXT.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-RESEARCH.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-01-SUMMARY.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-02-SUMMARY.md
@src/zeroth/core/examples/quickstart.py
@src/zeroth/core/policy/models.py
@src/zeroth/core/service/bootstrap.py
@src/zeroth/core/service/approval_api.py
@src/zeroth/core/service/audit_api.py
@src/zeroth/core/service/run_api.py
@src/zeroth/core/orchestrator/runtime.py
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write examples/governance_walkthrough.py</name>
  <files>
    examples/governance_walkthrough.py
  </files>
  <action>
    1. Read `src/zeroth/core/policy/models.py` to confirm the exact
       `Capability` enum values and `PolicyDefinition` constructor args.
       Read `src/zeroth/core/service/audit_api.py` for the timeline +
       audits endpoint shapes. Read `src/zeroth/core/graph/models.py`
       `NodeBase.policy_bindings` to confirm it's a list of policy_id
       strings bound to nodes.

    2. Create `examples/governance_walkthrough.py`. Structure:

       ```
       """Governance Walkthrough — approval gate, auditor, policy block."""

       def main() -> int:
           if not os.environ.get("OPENAI_API_KEY"):
               print("SKIP: ...", file=sys.stderr)
               return 0

           # Bootstrap one in-process service with SQLite
           app = bootstrap_service(...)

           # ── Scenario 1: Approval gate ───────────────────────────
           graph_approval = build_demo_graph(include_approval=True)
           deploy_approval_graph(...)
           run_id = submit_run(...)
           poll_until(paused_for_approval)
           approval_id = fetch_approval(run_id)
           resolve_approval(approval_id, "approve")
           poll_until(succeeded)
           print("Scenario 1 — approval gate: run succeeded after human approval")

           # ── Scenario 2: Auditor ─────────────────────────────────
           timeline = GET /runs/{run_id}/timeline
           for entry in timeline.entries:
               print(f"  [{entry.node_id}] {entry.status} — policy: {entry.policy_decisions}")
           print("Scenario 2 — auditor: full trail printed above")

           # ── Scenario 3: Policy block ────────────────────────────
           block_policy = PolicyDefinition(
               policy_id="block-network-write",
               denied_capabilities=[Capability.NETWORK_WRITE],
           )
           graph_blocked = build_demo_graph_with_policy(
               denied_capabilities=[Capability.NETWORK_WRITE],
           )
           register_policy(block_policy)
           deploy_blocked_graph(...)
           run_id_2 = submit_run(...)
           poll_until(terminated_by_policy)
           audits = GET /deployments/{ref}/audits?run_id=...
           print("Scenario 3 — policy block:")
           for rec in audits:
               if rec.policy_decision == "deny":
                   print(f"  DENIED at {rec.node_id}: {rec.reason}")
           return 0
       ```

    3. Use httpx `AsyncClient(app=app, base_url="http://test")` the same
       way `examples/approval_demo.py` does — NOT raw repository calls.
       This keeps the example showing the real HTTP surface a reader
       would use.

    4. Handle the reality gap honestly: if the runtime today does not
       actually enforce `Capability.NETWORK_WRITE` denial for the tool
       node shape returned by `build_demo_graph_with_policy`, the
       executor must either:
         (a) Use a different capability/node combo that IS enforced
             today (grep `orchestrator/runtime.py` for
             `RunStatus.TERMINATED_BY_POLICY` to find the enforcement
             path), OR
         (b) Extend `build_demo_graph_with_policy` minimally to produce
             a node shape that the current runtime will deny, AND
             update `src/zeroth/core/examples/quickstart.py` + its tests
             accordingly. Document the choice in SUMMARY.md.
       DO NOT fake the termination — the whole point of DOCS-05 is that
       the walkthrough runs end-to-end against the real runtime.

    5. SKIP guard on OPENAI_API_KEY identical to other examples. Exit 0
       on SKIP, exit 0 on success, exit non-zero only on genuine bugs.

    6. `uv run python examples/governance_walkthrough.py` locally —
       SKIP path must exit 0. If you have a key, the happy path must
       print all three scenario summaries without traceback.

    7. `uv run ruff check examples/governance_walkthrough.py` and format.
  </action>
  <verify>
    <automated>uv run python examples/governance_walkthrough.py &amp;&amp; uv run ruff check examples/governance_walkthrough.py</automated>
  </verify>
  <done>
    - Script runs all three scenarios against one bootstrap
    - SKIP path exits 0 cleanly
    - Uses httpx AsyncClient against the in-process FastAPI app (real HTTP surface)
    - References the real `/timeline` and `/audits` endpoints
    - Uses real `PolicyDefinition` + `Capability` enum values
    - Ruff clean
  </done>
</task>

<task type="auto">
  <name>Task 2: Write docs/tutorials/governance-walkthrough.md + shape tests</name>
  <files>
    docs/tutorials/governance-walkthrough.md
    tests/test_docs_phase30.py
  </files>
  <action>
    1. Rewrite `docs/tutorials/governance-walkthrough.md` (replacing the
       plan-02 placeholder) with this structure:

       - **H1: Governance Walkthrough**
       - Short intro: "This tutorial exercises three Zeroth
         differentiators in a single end-to-end run: an approval gate
         that pauses execution for human review, an auditor that makes
         every node's decisions inspectable, and a policy that blocks a
         tool call before it executes."
       - **Why this matters** section: one paragraph contrasting with
         LangGraph/CrewAI which ship agents but not governance
         primitives. Name-check the three Zeroth subsystems: approvals,
         audit, policy.
       - **Prerequisites** section: link back to Getting Started
         Install + First graph.
       - **Running the walkthrough**: one-line `uv run python
         examples/governance_walkthrough.py` + an admonition about the
         OPENAI_API_KEY requirement.
       - **Scenario 1 — Approval gate**: 2-3 paragraphs explaining the
         HumanApprovalNode, the `paused_for_approval` status, and the
         resolve endpoint. Embed a range of `examples/governance_walkthrough.py`
         if range markers are added, or embed the whole file once at the
         end if ranges are too fiddly.
       - **Scenario 2 — Auditor**: explain `GET /runs/{id}/timeline`,
         what a NodeAuditRecord contains, and what "audit trail"
         actually means in Zeroth (per-node, not monolithic logs).
       - **Scenario 3 — Policy block**: explain the `PolicyDefinition`,
         `denied_capabilities`, `policy_bindings` on nodes, and what
         `TERMINATED_BY_POLICY` looks like in the run lifecycle. Show
         the audit record that gets emitted.
       - **Full example** section at the bottom: `--8<-- "governance_walkthrough.py"`
         embed (whole file).
       - **Where to next**: link forward to the Phase 31 subsystem
         Concept pages for approvals/audit/policy (note these don't
         exist yet and link to an anchor that plan-02's stubs exposed).

    2. Extend `tests/test_docs_phase30.py`:
         * `test_governance_walkthrough_page_shape`: assert file exists,
           has H1 "Governance Walkthrough", and mentions all three
           keywords "approval", "audit", "policy" (case-insensitive).
         * `test_governance_walkthrough_embeds_example`: assert the page
           contains `--8<--` with `governance_walkthrough.py`.
         * `test_governance_walkthrough_example_covers_three_scenarios`:
           read `examples/governance_walkthrough.py`, assert substrings
           `approval`, `timeline`, and `Capability` ALL present (weak
           but honest check — the real validation is running the file).
         * `test_governance_walkthrough_example_skips_cleanly`: use
           `subprocess.run(["python", "examples/governance_walkthrough.py"],
           env={"PATH": ...})` WITHOUT `OPENAI_API_KEY`, assert
           `returncode == 0` and "SKIP" in stderr. Mark as
           `@pytest.mark.slow` if it takes more than a few seconds.

    3. Run `uv run pytest tests/test_docs_phase30.py -v` — green.
    4. Run `uv run mkdocs build --strict` — must pass (snippet must
       resolve).
    5. `uv run ruff check tests/test_docs_phase30.py` and format.
  </action>
  <verify>
    <automated>uv run pytest tests/test_docs_phase30.py -v &amp;&amp; uv run mkdocs build --strict</automated>
  </verify>
  <done>
    - `docs/tutorials/governance-walkthrough.md` has all five sections + full example embed
    - Page explains each scenario in prose that matches the script's actual behavior
    - Four new shape tests green
    - Strict mkdocs build green
  </done>
</task>

</tasks>

<verification>
- `uv run python examples/governance_walkthrough.py` → SKIP (or success end-to-end), exit 0
- `uv run mkdocs build --strict` → green
- `uv run pytest tests/test_docs_phase30.py -v` → all tests green
- Manual spot-check: the prose on each scenario matches what the script actually does (no lies)
</verification>

<success_criteria>
- DOCS-05 shipped: one tutorial page + one runnable example exercising approval gate + auditor + policy block
- Example uses real `PolicyDefinition` / `Capability` / `/timeline` / `/audits` / `/approvals/.../resolve` — no fakes
- Page is substantive (not a toy) and frames the three scenarios as Zeroth's differentiators
- Script exit code 0 on SKIP path; the examples.yml CI workflow from plan 03 will pick this file up automatically via its `[ -f ]` guard
</success_criteria>

<output>
After completion, create `.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-04-SUMMARY.md` documenting: which capability/node combination actually triggers TERMINATED_BY_POLICY in the current runtime, any changes made to `zeroth.core.examples.quickstart.build_demo_graph_with_policy`, and the full command used to verify the happy path if the executor had an OPENAI_API_KEY available.
</output>
