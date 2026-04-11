---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 05
subsystem: docs-cookbook
tags: [docs, cookbook, examples, nav, mkdocs-strict, diataxis]
requires: [31-01, 31-02, 31-03, 31-04]
provides:
  - docs/how-to/cookbook/ (index + 10 recipes)
  - examples/*.py (10 runnable files)
  - mkdocs.yml full nav
  - .github/workflows/examples.yml extended CI matrix
  - meaningful docs/concepts/index.md and docs/how-to/index.md
  - green `mkdocs build --strict`
affects:
  - docs/how-to/cookbook/
  - docs/concepts/index.md
  - docs/how-to/index.md
  - examples/
  - mkdocs.yml
  - .github/workflows/examples.yml
tech-stack:
  added: []
  patterns:
    - "pymdownx.snippets embedding examples/*.py into cookbook recipe pages (base_path=[.,examples])"
    - "SKIP-on-missing-env shape for every new example file"
    - "In-process deterministic stubs instead of LLM calls (required_env=[])"
    - "Diataxis cross-linking: cookbook -> usage guide + concept pages"
key-files:
  created:
    - examples/approval_step.py
    - examples/attach_memory.py
    - examples/budget_cap.py
    - examples/sandbox_tool.py
    - examples/webhook_retry.py
    - examples/policy_block.py
    - examples/audit_query.py
    - examples/agent_handoff.py
    - examples/condition_branch.py
    - examples/secret_injection.py
    - docs/how-to/cookbook/index.md
    - docs/how-to/cookbook/approval-step.md
    - docs/how-to/cookbook/attach-memory.md
    - docs/how-to/cookbook/budget-cap.md
    - docs/how-to/cookbook/sandbox-tool.md
    - docs/how-to/cookbook/webhook-retry.md
    - docs/how-to/cookbook/policy-block.md
    - docs/how-to/cookbook/audit-query.md
    - docs/how-to/cookbook/agent-handoff.md
    - docs/how-to/cookbook/condition-branch.md
    - docs/how-to/cookbook/secret-injection.md
  modified:
    - docs/concepts/index.md
    - docs/how-to/index.md
    - mkdocs.yml
    - .github/workflows/examples.yml
key-decisions:
  - "All 10 new examples use required_env=[] — no LLM credentials needed, which keeps forked-PR CI green and lets readers run the cookbook offline. Agent-handoff uses deterministic stubs instead of litellm."
  - "Cookbook recipe pages embed snippets via `--8<-- \"<slug>.py\"` (no `examples/` prefix) because mkdocs.yml already sets pymdownx.snippets.base_path = [\".\", \"examples\"]."
  - "Concepts index rewrite groups the 20 subsystems into Execution / Data / Governance / Platform with the same taxonomy the how-to index uses, so the two landing pages mirror each other."
  - "policy_block.py registers every Capability value explicitly (CapabilityRegistry is NOT self-populating — called out in 31-03 summary and reused here)."
  - "approval_step.py passes an ActorIdentity to ApprovalService.resolve because actor is a required keyword arg — the examples in plan 30 used the HTTP endpoint which handles actor construction internally."
requirements-completed:
  - DOCS-06
  - DOCS-12
duration: 11 min
completed: 2026-04-11
---

# Phase 31 Plan 05: Cookbook + Examples + Nav Finalize Summary

**One-liner:** Shipped 10 runnable cookbook examples (all in-process,
all SKIP-safe, zero-LLM), 11 cookbook markdown pages with live snippet
embeds, rewrote both Diátaxis landing pages, wired the full 40-page +
10-recipe nav in `mkdocs.yml`, extended the CI matrix to cover every
new example, and brought the site through `uv run mkdocs build --strict`
with exit 0.

## Overview

- **Plan:** 31-05 Cookbook + Examples + Nav Finalize
- **Duration:** ~11 min
- **Start:** 2026-04-11T20:52:43Z
- **End:** 2026-04-11T21:03:27Z
- **Tasks:** 5/5 complete
- **Files created:** 21 (10 examples + 11 cookbook markdown)
- **Files modified:** 4 (concepts/index, how-to/index, mkdocs.yml, examples.yml)

## What shipped

### 10 example files (all `required_env = []`, exit 0 clean)

| File | Subsystems exercised | Runs against |
|------|---------------------|--------------|
| `examples/approval_step.py` | approvals, graph, runs | in-process SQLite + bootstrap_service |
| `examples/attach_memory.py` | memory, agents | `RunEphemeralMemoryConnector` in-process |
| `examples/budget_cap.py` | econ, runs | `CostEstimator` offline (litellm pricing) |
| `examples/sandbox_tool.py` | execution_units, guardrails | `SandboxManager` local-subprocess backend |
| `examples/webhook_retry.py` | webhooks, dispatch | in-process (`sign_payload`, `next_retry_delay`) |
| `examples/policy_block.py` | policy, execution_units | `PolicyGuard` in-process |
| `examples/audit_query.py` | audit, runs | in-process SQLite (`AuditRepository`) |
| `examples/agent_handoff.py` | agents, orchestrator | in-process SQLite + deterministic stub runners |
| `examples/condition_branch.py` | conditions, graph | `BranchResolver` in-process |
| `examples/secret_injection.py` | secrets, execution_units | `EnvSecretProvider` + `SecretRedactor` in-process |

**Smoke-test evidence:** each file was run once immediately after it
was written; every file exited 0. Final sequential run of all 10 after
commit also exited 0. `uv run ruff check examples/approval_step.py …
examples/secret_injection.py` → `All checks passed!`.

### 10 cookbook recipe pages + 1 index

| Recipe page | Primary subsystem links | Embeds |
|-------------|------------------------|--------|
| `docs/how-to/cookbook/approval-step.md` | approvals, graph | `approval_step.py` |
| `docs/how-to/cookbook/attach-memory.md` | memory, agents | `attach_memory.py` |
| `docs/how-to/cookbook/budget-cap.md` | econ, runs | `budget_cap.py` |
| `docs/how-to/cookbook/sandbox-tool.md` | execution-units, guardrails | `sandbox_tool.py` |
| `docs/how-to/cookbook/webhook-retry.md` | webhooks, dispatch | `webhook_retry.py` |
| `docs/how-to/cookbook/policy-block.md` | policy, execution-units | `policy_block.py` |
| `docs/how-to/cookbook/audit-query.md` | audit, runs | `audit_query.py` |
| `docs/how-to/cookbook/agent-handoff.md` | agents, orchestrator | `agent_handoff.py` |
| `docs/how-to/cookbook/condition-branch.md` | conditions, graph | `condition_branch.py` |
| `docs/how-to/cookbook/secret-injection.md` | secrets, execution-units | `secret_injection.py` |

Every recipe has exactly the six required H2 sections
(`What this recipe does`, `When to use`, `When NOT to use`, `Recipe`,
`How it works`, `See also`) and embeds its matching example via
`--8<-- "<slug>.py"`.

### mkdocs.yml nav diff summary

- **Removed** the `- how-to/index.md` and `- concepts/index.md`
  placeholder-only entries.
- **Added** under `How-to Guides`: a `Subsystems:` subsection listing
  all 20 usage-guide pages and a `Cookbook:` subsection listing
  `cookbook/index.md` + all 10 recipe pages.
- **Added** under `Concepts`: every one of the 20 concept pages as
  top-level siblings of `concepts/index.md`.
- Tutorials and Reference sections were left untouched.

Net change: **+53 lines, −0 lines** in the nav block.

### .github/workflows/examples.yml diff summary

- Appended 10 new `- name: Run examples/<file>.py` steps after the
  existing `governance_walkthrough` step, one per new file.
- Existing 4 steps (`hello`, `first_graph`, `approval_demo`,
  `governance_walkthrough`) were preserved verbatim.
- Env forwarding for `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` left
  unchanged; the SKIP pattern on every new file means forked-PR
  runs without repo secrets still exit 0 (actually, every new
  example has `required_env = []` so SKIP is unreachable for them).

Net change: **+30 lines, −0 lines**.

### Rewritten landing pages

- `docs/concepts/index.md`: from a 3-line stub to a categorized
  landing page grouping all 20 concept pages into **Execution**,
  **Data and state**, **Governance**, **Platform**, and closing with
  an explicit note on the threads → webhooks substitution.
- `docs/how-to/index.md`: from a 3-line stub to a parallel landing
  page with the same 4 category groupings plus a dedicated
  **Cookbook** section pointing at `cookbook/index.md`.

## Strict build gate result

```
$ uv run mkdocs build --strict
INFO    -  Cleaning site directory
INFO    -  Building documentation to directory: /Users/dondoe/coding/zeroth/site
INFO    -  Doc file 'how-to/agents.md' contains a link '../reference/python-api.md#zerothcoreagent_runtime', but the doc 'reference/python-api.md' does not contain an anchor '#zerothcoreagent_runtime'.
…(11 similar INFO anchor notes, all pointing at reference/python-api.md#<anchor>)…
INFO    -  Documentation built in 0.59 seconds
$ echo $?
0
```

**Exit code 0.** The 11 `INFO` lines are not strict-mode errors — they
are informational notes about forward-reference anchors in
`reference/python-api.md` that Phase 32 will populate with mkdocstrings
output. Under mkdocs strict mode, only `WARNING` and above cause the
build to fail; `INFO` anchor notes do not. Built 11 cookbook pages, 20
concept pages, 20 usage-guide pages, and all the existing
tutorials/reference content into `site/`.

Spot-checked `site/how-to/cookbook/approval-step/index.html` — the
embedded Python snippet from `examples/approval_step.py` is present
verbatim in the rendered HTML (`grep` for `_EchoAgentRunner` → hit).

## Notes for Phase 32 (Reference quadrant)

These 11 forward-reference anchors are the first thing Phase 32 should
address — they exist as `INFO`-level notes today but a stricter build
config (or a future mkdocs version) could escalate them to WARNINGs.
They live in:

- `docs/how-to/agents.md` → `#zerothcoreagent_runtime`
- `docs/how-to/approvals.md` → `#approvals`
- `docs/how-to/audit.md` → `#audit`
- `docs/how-to/conditions.md` → `#zerothcoreconditions`
- `docs/how-to/execution-units.md` → `#zerothcoreexecution_units`
- `docs/how-to/graph.md` → `#zerothcoregraph`
- `docs/how-to/guardrails.md` → `#guardrails`
- `docs/how-to/identity.md` → `#identity`, `#service-auth`
- `docs/how-to/orchestrator.md` → `#zerothcoreorchestrator`
- `docs/how-to/policy.md` → `#policy`

Phase 32's mkdocstrings wiring should produce anchors matching the
bare module name (`zerothcore<subsystem>`) — the batch A/B/C usage
guides already assume that convention. Batch C (governance) used
short forms like `#approvals` / `#audit` / `#policy` / `#guardrails` /
`#identity` — Phase 32 should either rewrite those links to the
`zerothcore<subsystem>` form or generate both anchor variants.

## Example-file API notes (for future example-file cleanup)

While writing the 10 new examples I had to work around a few
gotchas already flagged in earlier summaries:

1. **`ApprovalService.resolve` needs `actor: ActorIdentity`.** Not
   optional. Previous examples resolved approvals via the HTTP
   endpoint which synthesizes the actor from the authenticated
   principal. `approval_step.py` constructs an `ActorIdentity`
   directly.
2. **`CapabilityRegistry` is not self-populating.** `policy_block.py`
   registers every `Capability` value in a loop — matches the
   `examples/governance_walkthrough.py` pattern.
3. **`BudgetEnforcer` requires a Regulus backend.** To keep
   `budget_cap.py` offline, the recipe builds the gate from the
   offline `CostEstimator` directly and references `BudgetEnforcer`
   as "the online cousin" in the cookbook page prose.
4. **`ConnectorManifest` shape.** Not `memory_ref=...` — that field
   is the registry key, passed separately to
   `InMemoryConnectorRegistry.register(memory_ref, manifest, connector)`.
   The manifest itself takes `connector_type`, `scope`, optional
   `instance_id`, and `config`.
5. **`SandboxManager.run` takes a `timeout_seconds` kwarg** (not
   `timeout`) and returns a `SandboxExecutionResult` dataclass with
   `returncode`, `stdout`, `stderr`, `backend`, etc.

## Deviations from Plan

None — plan executed exactly as written. All five tasks completed in
order, every verification gate passed on the first attempt.

- **Rule 1 (Bug):** none
- **Rule 2 (Missing critical):** none
- **Rule 3 (Blocking):** none
- **Rule 4 (Architectural):** none

**Total deviations:** 0.

## Verification

- `uv run ruff check examples/<10 new files>` → `All checks passed!`
- 10 examples × 1 run each → all exit 0
- `ls docs/how-to/cookbook/ | wc -l` → 11
- Every recipe has all 6 H2 sections (What / When to use / When NOT /
  Recipe / How it works / See also) — verified at write time via
  template adherence
- `uv run mkdocs build --strict` → exit 0, built in 0.59s
- `site/how-to/cookbook/approval-step/index.html` contains the
  embedded `_EchoAgentRunner` class → snippet embedding confirmed
- mkdocs.yml nav lists all 20 concepts, all 20 usage guides, all 10
  cookbook recipes + cookbook/index.md
- `.github/workflows/examples.yml` contains all 14 example files (4
  existing + 10 new) — confirmed via grep on each filename

## Commits

| Task | Commit | Files | Lines |
|------|--------|-------|-------|
| 1 — 10 example files | `63b64d2` | 10 new `.py` files | +~700 |
| 2 — cookbook markdown | `e1b6871` | 11 new `.md` files | +~330 |
| 3 — landing pages | `f481186` | 2 modified index pages | +133 / −2 |
| 4 — nav + CI | `f833f81` | mkdocs.yml + examples.yml | +83 / −0 |

## Issues Encountered

None.

## Next Phase Readiness

**Phase 31 complete.** With 31-05 shipped, Phase 31 has delivered:

- 20 Concept pages (5 per batch, 4 batches)
- 20 Usage Guide pages (same breakdown)
- 10 Cookbook recipe pages + index
- 10 runnable example files
- 2 meaningful Diátaxis landing pages (concepts + how-to)
- Full mkdocs.yml nav covering all 41 new pages
- CI matrix running all 14 example files on every main commit
- Green `uv run mkdocs build --strict`

Phase 31 requirements DOCS-03, DOCS-04, DOCS-06, DOCS-12 are all
satisfied. Ready for Phase 32 (Reference quadrant: mkdocstrings Python
API, FastAPI HTTP API, configuration reference, deployment guide,
migration guide).

## Self-Check: PASSED

- examples/approval_step.py: FOUND
- examples/attach_memory.py: FOUND
- examples/budget_cap.py: FOUND
- examples/sandbox_tool.py: FOUND
- examples/webhook_retry.py: FOUND
- examples/policy_block.py: FOUND
- examples/audit_query.py: FOUND
- examples/agent_handoff.py: FOUND
- examples/condition_branch.py: FOUND
- examples/secret_injection.py: FOUND
- docs/how-to/cookbook/index.md: FOUND
- docs/how-to/cookbook/approval-step.md: FOUND
- docs/how-to/cookbook/attach-memory.md: FOUND
- docs/how-to/cookbook/budget-cap.md: FOUND
- docs/how-to/cookbook/sandbox-tool.md: FOUND
- docs/how-to/cookbook/webhook-retry.md: FOUND
- docs/how-to/cookbook/policy-block.md: FOUND
- docs/how-to/cookbook/audit-query.md: FOUND
- docs/how-to/cookbook/agent-handoff.md: FOUND
- docs/how-to/cookbook/condition-branch.md: FOUND
- docs/how-to/cookbook/secret-injection.md: FOUND
- commit 63b64d2 (Task 1 examples): FOUND
- commit e1b6871 (Task 2 cookbook pages): FOUND
- commit f481186 (Task 3 landing pages): FOUND
- commit f833f81 (Task 4 nav + CI): FOUND
