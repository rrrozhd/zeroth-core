---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 03
type: execute
wave: 1
depends_on: []
files_modified:
  - docs/concepts/policy.md
  - docs/concepts/approvals.md
  - docs/concepts/audit.md
  - docs/concepts/guardrails.md
  - docs/concepts/identity.md
  - docs/how-to/policy.md
  - docs/how-to/approvals.md
  - docs/how-to/audit.md
  - docs/how-to/guardrails.md
  - docs/how-to/identity.md
autonomous: true
requirements:
  - DOCS-03
  - DOCS-04
subsystem_map:
  policy: src/zeroth/core/policy/
  approvals: src/zeroth/core/approvals/
  audit: src/zeroth/core/audit/
  guardrails: src/zeroth/core/guardrails/
  identity: src/zeroth/core/identity/

must_haves:
  truths:
    - "Reader can understand Zeroth's governance story: policy blocks, approvals gate, audit records, guardrails constrain, identity authenticates"
    - "Each Concept page for this batch cross-links to at least one other governance subsystem in the batch"
    - "policy Usage Guide shows a rule that blocks a specific tool call"
    - "approvals Usage Guide shows attaching an approval gate to a node (match existing examples/approval_demo.py pattern)"
    - "audit Usage Guide shows querying audit entries for a given run_id"
  artifacts:
    - path: docs/concepts/policy.md
      min_lines: 40
      contains: "## What it is"
    - path: docs/concepts/approvals.md
      min_lines: 40
    - path: docs/concepts/audit.md
      min_lines: 40
    - path: docs/concepts/guardrails.md
      min_lines: 40
    - path: docs/concepts/identity.md
      min_lines: 40
    - path: docs/how-to/policy.md
      min_lines: 50
      contains: "## Minimal example"
    - path: docs/how-to/approvals.md
      min_lines: 50
    - path: docs/how-to/audit.md
      min_lines: 50
    - path: docs/how-to/guardrails.md
      min_lines: 50
    - path: docs/how-to/identity.md
      min_lines: 50
  key_links:
    - from: docs/how-to/approvals.md
      to: examples/approval_demo.py
      via: "minimal example reuses existing approval_demo.py pattern"
      pattern: "approval_demo|enqueue_approval|decide"
    - from: docs/concepts/audit.md
      to: docs/concepts/runs.md
      via: "'Where it fits' — audit entries are attached to runs"
      pattern: "runs.md|concepts/runs"
---

<objective>
Ship 10 docs pages covering the governance slice of Zeroth: policy, approvals, audit, guardrails, identity. One Concept + one Usage Guide per subsystem.

Purpose: This is Zeroth's core differentiator vs LangGraph/CrewAI. The pages must make the governance value crystal clear. Content is synthesized from the actual modules under `src/zeroth/core/`.
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

@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-04-governance-walkthrough-tutorial-SUMMARY.md

Source subsystems:
@src/zeroth/core/policy/
@src/zeroth/core/approvals/
@src/zeroth/core/audit/
@src/zeroth/core/guardrails/
@src/zeroth/core/identity/

Canonical example patterns to reuse (DO NOT invent new APIs):
@examples/approval_demo.py
@examples/governance_walkthrough.py

Existing tutorial that tells the end-to-end governance story — link to it from "See also" in Concept pages:
@docs/tutorials/governance-walkthrough.md

<page_templates>
Concept page (~300 words, 5 required H2 sections): `## What it is`, `## Why it exists`, `## Where it fits`, `## Key types`, `## See also`.

Usage Guide (~400-500 words, 5 required H2 sections): `## Overview`, `## Minimal example`, `## Common patterns`, `## Pitfalls`, `## Reference cross-link`.

Filename rule: single-word slugs matching module names.

Governance-specific rule: every Concept page's "See also" section MUST link to `../tutorials/governance-walkthrough.md` so readers can go from a single-subsystem Concept straight to the end-to-end story.
</page_templates>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Concept + Usage Guide for policy, approvals, audit</name>
  <files>
    docs/concepts/policy.md,
    docs/concepts/approvals.md,
    docs/concepts/audit.md,
    docs/how-to/policy.md,
    docs/how-to/approvals.md,
    docs/how-to/audit.md
  </files>
  <action>
    For each of policy, approvals, audit:

    1. Read src/zeroth/core/{module}/ thoroughly — the Zeroth governance story is codified here, so extract terminology faithfully.

    2. Write docs/concepts/{slug}.md using the Concept template. Cross-link these three to each other in "Where it fits" (policy blocks before execution, approvals gate at a node, audit records the outcome). "See also" MUST include ../tutorials/governance-walkthrough.md.

    3. Write docs/how-to/{slug}.md:
       - policy: Minimal example shows a rule blocking a specific tool call (read examples/governance_walkthrough.py for the real API).
       - approvals: Minimal example matches examples/approval_demo.py — prefer embedding a 10-20 line slice via pymdownx.snippets if practical, otherwise copy verbatim. Must show enqueue + decide.
       - audit: Minimal example queries the audit trail for a completed run_id.

    4. All snippets must be runnable with zeroth.core.* imports. Before writing API calls, verify they exist in the source — if the real API differs from the example files, follow the source, not guesses.

    Do NOT touch mkdocs.yml. Do NOT modify index.md files.
  </action>
  <verify>
    <automated>for f in docs/concepts/policy.md docs/concepts/approvals.md docs/concepts/audit.md docs/how-to/policy.md docs/how-to/approvals.md docs/how-to/audit.md; do test -f "$f" || exit 1; done && grep -l "governance-walkthrough" docs/concepts/policy.md docs/concepts/approvals.md docs/concepts/audit.md</automated>
  </verify>
  <done>Six files exist. Every Concept "See also" links to the governance walkthrough tutorial. approvals Usage Guide example matches the enqueue/decide pattern from approval_demo.py.</done>
</task>

<task type="auto">
  <name>Task 2: Concept + Usage Guide for guardrails, identity</name>
  <files>
    docs/concepts/guardrails.md,
    docs/concepts/identity.md,
    docs/how-to/guardrails.md,
    docs/how-to/identity.md
  </files>
  <action>
    For guardrails and identity:

    1. Read src/zeroth/core/guardrails/ and src/zeroth/core/identity/. Guardrails covers runtime constraints (rate limits, output validation, judges); identity covers tenant/user authentication and RBAC.

    2. Write docs/concepts/guardrails.md and docs/concepts/identity.md. Cross-link guardrails to policy (both are protective layers); cross-link identity to approvals (approvals authenticate the decider) and service (auth enforced at API boundary). "See also" in both MUST link to ../tutorials/governance-walkthrough.md.

    3. Write docs/how-to/guardrails.md:
       - Minimal example: attach a guardrail to a node or a run
       - Common patterns: rate limiting, schema validation, judge integration
       - Pitfalls: guardrail ordering, fallback behaviour

    4. Write docs/how-to/identity.md:
       - Minimal example: create a session or configure an auth provider (check src/zeroth/core/identity/ and src/zeroth/core/service/auth.py for the real API)
       - Common patterns: tenant scoping, role checks
       - Pitfalls: token expiry, missing claims, tenant leakage

    Do NOT touch mkdocs.yml. Do NOT modify index.md files.
  </action>
  <verify>
    <automated>test -f docs/concepts/guardrails.md && test -f docs/concepts/identity.md && test -f docs/how-to/guardrails.md && test -f docs/how-to/identity.md && grep -q "governance-walkthrough" docs/concepts/guardrails.md && grep -q "governance-walkthrough" docs/concepts/identity.md</automated>
  </verify>
  <done>Four files exist with all required sections. guardrails cross-links to policy; identity cross-links to approvals and to service/auth.py concepts.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build` (non-strict) succeeds
- All 10 files exist; each Concept has 5 required H2 sections; each Usage Guide has 5
- Every Concept page in this batch links to ../tutorials/governance-walkthrough.md in "See also"
- approvals Usage Guide example mirrors examples/approval_demo.py (enqueue + decide shape)
- No legacy `zeroth.*` imports anywhere in new files
</verification>

<success_criteria>
- 10 new markdown files committed
- Governance story is coherent across the 5 pages: a reader can follow policy -> approvals -> audit in-page links without bouncing
- mkdocs build passes
</success_criteria>

<output>
After completion, create `.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-03-SUMMARY.md` listing files shipped plus any API surprises (governance APIs evolved across phases; note anything that diverges from examples/).
</output>
