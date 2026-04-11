---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 04
type: execute
wave: 1
depends_on: []
files_modified:
  - docs/concepts/secrets.md
  - docs/concepts/dispatch.md
  - docs/concepts/econ.md
  - docs/concepts/service.md
  - docs/concepts/webhooks.md
  - docs/how-to/secrets.md
  - docs/how-to/dispatch.md
  - docs/how-to/econ.md
  - docs/how-to/service.md
  - docs/how-to/webhooks.md
autonomous: true
requirements:
  - DOCS-03
  - DOCS-04
subsystem_map:
  secrets: src/zeroth/core/secrets/
  dispatch: src/zeroth/core/dispatch/
  econ: src/zeroth/core/econ/
  service: src/zeroth/core/service/
  webhooks: src/zeroth/core/webhooks/
subsystem_substitution:
  original: threads
  substitute: webhooks
  reason: "No zeroth.core.threads module exists in the tree. Per CONTEXT D-01 + Claude's Discretion, substituting with a concrete user-facing subsystem. `webhooks` is the best fit — it is a user-facing async integration surface and has no coverage otherwise. `deployments` is a closer structural match but is too operational for the Concepts quadrant; it belongs in the Phase 32 Deployment Guide."

must_haves:
  truths:
    - "Reader can understand the platform slice of Zeroth: how secrets are stored, how the service boots, how work is dispatched, how economics are tracked, how webhooks fire"
    - "econ Concept page clarifies the Regulus/econ-instrumentation-sdk relationship"
    - "dispatch Usage Guide mentions the [dispatch] extra (redis + arq)"
    - "service Usage Guide shows the uvicorn/entrypoint.py bootstrap"
    - "webhooks Usage Guide covers retry + DLQ behaviour"
    - "Batch spec's 20th subsystem (threads) is substituted with webhooks; substitution is documented in frontmatter"
  artifacts:
    - path: docs/concepts/secrets.md
      min_lines: 40
      contains: "## What it is"
    - path: docs/concepts/dispatch.md
      min_lines: 40
    - path: docs/concepts/econ.md
      min_lines: 40
    - path: docs/concepts/service.md
      min_lines: 40
    - path: docs/concepts/webhooks.md
      min_lines: 40
    - path: docs/how-to/secrets.md
      min_lines: 50
      contains: "## Minimal example"
    - path: docs/how-to/dispatch.md
      min_lines: 50
    - path: docs/how-to/econ.md
      min_lines: 50
    - path: docs/how-to/service.md
      min_lines: 50
    - path: docs/how-to/webhooks.md
      min_lines: 50
  key_links:
    - from: docs/how-to/dispatch.md
      to: "pyproject.toml [dispatch] extra"
      via: "install command"
      pattern: "zeroth-core\\[dispatch\\]|dispatch.*redis|arq"
    - from: docs/concepts/econ.md
      to: "econ-instrumentation-sdk / Regulus"
      via: "pedagogical explanation of the external companion"
      pattern: "econ-instrumentation-sdk|Regulus"
---

<objective>
Ship 10 docs pages covering the platform slice of Zeroth: secrets, dispatch, econ, service, webhooks. One Concept + one Usage Guide per subsystem.

Purpose: Complete subsystem coverage. The CONTEXT spec lists `threads` as the 20th subsystem, but no `zeroth.core.threads` module exists. Per CONTEXT "Claude's Discretion", we substitute with `webhooks` — a concrete, user-facing, async integration surface that otherwise has no documentation coverage. The substitution is recorded in this plan's frontmatter for auditability.
Output: 10 new markdown files. No mkdocs.yml changes — plan 31-05 handles nav.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-CONTEXT.md

Source subsystems:
@src/zeroth/core/secrets/
@src/zeroth/core/dispatch/
@src/zeroth/core/econ/
@src/zeroth/core/service/
@src/zeroth/core/webhooks/

Relevant service entrypoints:
@src/zeroth/core/service/entrypoint.py
@src/zeroth/core/service/app.py
@src/zeroth/core/service/bootstrap.py

Extras declarations (for install commands):
@pyproject.toml

<page_templates>
Concept page (~300 words, 5 required H2 sections): `## What it is`, `## Why it exists`, `## Where it fits`, `## Key types`, `## See also`.
Usage Guide (~400-500 words, 5 required H2 sections): `## Overview`, `## Minimal example`, `## Common patterns`, `## Pitfalls`, `## Reference cross-link`.
</page_templates>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Concept + Usage Guide for secrets, econ, service</name>
  <files>
    docs/concepts/secrets.md,
    docs/concepts/econ.md,
    docs/concepts/service.md,
    docs/how-to/secrets.md,
    docs/how-to/econ.md,
    docs/how-to/service.md
  </files>
  <action>
    For each of secrets, econ, service:

    1. Read src/zeroth/core/{module}/ and identify 3-5 key public types.

    2. Write docs/concepts/{slug}.md:
       - secrets: explain the provider abstraction (env vs vault etc.). Cross-link to concepts/identity.md and concepts/service.md.
       - econ: explain that Zeroth instruments cost via the external `econ-instrumentation-sdk` (Regulus). Cite the dependency by name as it appears in pyproject.toml. Cross-link to concepts/runs.md.
       - service: explain that Zeroth ships a FastAPI service (entrypoint.py + app.py + bootstrap.py). Cross-link to concepts/identity.md and concepts/orchestrator.md.

    3. Write docs/how-to/{slug}.md:
       - secrets: minimal example = load a secret via the provider interface; pitfalls = rotation, scoping, plaintext env risk.
       - econ: minimal example = instrument a node with a cost hook; common patterns = budget caps, unit types; pitfalls = SDK version skew, missing Regulus service.
       - service: minimal example = run `uvicorn zeroth.core.service.entrypoint:app` or show the entrypoint module name explicitly (verify path by reading the real file); common patterns = mounting under a prefix, healthchecks; pitfalls = startup ordering (identity -> storage -> dispatch -> service).

    Do NOT touch mkdocs.yml or index.md files.
  </action>
  <verify>
    <automated>for f in docs/concepts/secrets.md docs/concepts/econ.md docs/concepts/service.md docs/how-to/secrets.md docs/how-to/econ.md docs/how-to/service.md; do test -f "$f" || exit 1; done && grep -q "econ-instrumentation-sdk\|Regulus" docs/concepts/econ.md && grep -q "entrypoint\|uvicorn" docs/how-to/service.md</automated>
  </verify>
  <done>Six files exist with required sections. econ page cites the external Regulus SDK by its real pyproject.toml name. service page shows the real uvicorn entrypoint.</done>
</task>

<task type="auto">
  <name>Task 2: Concept + Usage Guide for dispatch and webhooks</name>
  <files>
    docs/concepts/dispatch.md,
    docs/concepts/webhooks.md,
    docs/how-to/dispatch.md,
    docs/how-to/webhooks.md
  </files>
  <action>
    For dispatch and webhooks:

    1. Read src/zeroth/core/dispatch/ (especially worker.py) and src/zeroth/core/webhooks/.

    2. Write docs/concepts/dispatch.md:
       - Explain the distributed work dispatcher and the Redis/arq backing
       - "Where it fits" cross-links to orchestrator (orchestrator submits work) and service (service hosts the API side)
       - Mention the `[dispatch]` extra at the Concept level too

    3. Write docs/concepts/webhooks.md:
       - Explain the outbound webhook system (signed deliveries, retries, DLQ)
       - Cross-link to concepts/runs.md (runs emit webhook events) and concepts/audit.md (delivery attempts are audited)
       - Add a clear note at the top: "Note: the phase 31 spec originally listed `threads` as the 20th subsystem. No `zeroth.core.threads` module exists in the current tree, so this slot is filled by `webhooks`, a concrete user-facing async subsystem." (or an equivalent line — this is required for transparency, and the planner recorded the substitution in plan frontmatter.)

    4. Write docs/how-to/dispatch.md:
       - Minimal example: enqueue a unit of work (verify the real API in worker.py before writing)
       - Common patterns: queue sharding, worker scale-out, priority
       - Pitfalls: Redis unavailability, visibility timeouts, at-least-once semantics
       - Install section: `pip install 'zeroth-core[dispatch]'` — verify extra name in pyproject.toml

    5. Write docs/how-to/webhooks.md:
       - Minimal example: register a webhook endpoint for a run event (verify real API in src/zeroth/core/webhooks/ and src/zeroth/core/service/webhook_api.py)
       - Common patterns: signing secrets, retry with backoff, DLQ replay
       - Pitfalls: timeout tuning, idempotency, signature verification on the receiver
       - Cross-link to how-to/dispatch.md — webhook retries often flow through dispatch

    Do NOT touch mkdocs.yml or index.md files.
  </action>
  <verify>
    <automated>test -f docs/concepts/dispatch.md && test -f docs/concepts/webhooks.md && test -f docs/how-to/dispatch.md && test -f docs/how-to/webhooks.md && grep -q "zeroth-core\[dispatch\]\|memory-pg\|dispatch" docs/how-to/dispatch.md && grep -iq "threads\|substitut" docs/concepts/webhooks.md</automated>
  </verify>
  <done>Four files exist. dispatch page documents the [dispatch] extra. webhooks page contains the substitution note. Both pages cross-link to runs/audit/orchestrator as appropriate.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build` (non-strict) succeeds
- All 10 files exist with required H2 structure
- webhooks Concept page contains the explicit substitution note (threads -> webhooks)
- dispatch page documents the [dispatch] extra install command
- econ page names the external `econ-instrumentation-sdk` / Regulus dependency
- Cross-links to other batches (concepts/runs.md, concepts/audit.md, concepts/identity.md, concepts/orchestrator.md) use relative paths
</verification>

<success_criteria>
- 10 new markdown files committed
- Substitution of webhooks for threads is auditable (frontmatter + in-page note)
- mkdocs build passes
- Batch D completes the 40-page Concept + Usage Guide set together with 31-01..03
</success_criteria>

<output>
After completion, create `.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-04-SUMMARY.md` listing the 10 files, the threads->webhooks substitution rationale, any surprises reading dispatch/worker.py (arq API specifics), and any real entrypoint path details needed by plan 31-05's cookbook.
</output>
