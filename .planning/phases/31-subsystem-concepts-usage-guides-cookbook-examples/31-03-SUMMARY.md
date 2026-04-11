---
phase: 31
plan: 03
subsystem: docs-governance-batch
tags: [docs, governance, policy, approvals, audit, guardrails, identity]
requires: [docs-site-foundation]
provides: [governance-subsystem-pages]
affects: [docs/concepts, docs/how-to]
tech-stack:
  added: []
  patterns: [diataxis-concept+howto, cross-linking]
key-files:
  created:
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
  modified: []
key-decisions:
  - Followed source-of-truth APIs over example file drift (example uses ServiceRole.AUDITOR which doesn't exist in src; docs stick to OPERATOR / REVIEWER / ADMIN)
  - Cross-linked governance story bidirectionally: policy → approvals → audit, plus guardrails→policy, identity→approvals/service
  - Every Concept page "See also" links to ../tutorials/governance-walkthrough.md as the end-to-end driver
  - approvals Usage Guide mirrors enqueue/decide shape of examples/approval_demo.py
  - policy Usage Guide reproduces the PolicyGuard wiring pattern from examples/governance_walkthrough.py
requirements-completed:
  - DOCS-03
  - DOCS-04
duration: 28 min
completed: 2026-04-11
---

# Phase 31 Plan 03: Subsystems Batch C (Governance) Summary

Shipped 10 pages covering Zeroth's governance slice — `policy`, `approvals`, `audit`, `guardrails`, `identity` — each with a paired Diataxis Concept (~300 words, 5 H2s) and Usage Guide (~400–500 words, 5 H2s). The five-page arc now tells the full pre-flight → gate → record story: `PolicyGuard` blocks illegal capabilities before execution, `HumanApprovalNode` pauses runs for human sign-off, `NodeAuditRecord` captures every step with hash-chained integrity, `TokenBucketRateLimiter`/`QuotaEnforcer` bound runtime volume, and `ActorIdentity`/`AuthenticatedPrincipal` stamp every record with tenant-scoped role context.

## Task Summary

| Task | Name | Files | Commit |
|------|------|-------|--------|
| 1 | Concept + Usage Guide for policy, approvals, audit | 6 | `7ec79bb` |
| 2 | Concept + Usage Guide for guardrails, identity | 4 | `a40d594` |

**Duration:** ~28 min. **Tasks:** 2/2. **Files created:** 10.

## Verification

- `uv run mkdocs build` → succeeds with **0 link warnings** after stripping one preemptive `deployments.md` link (that concept page is owned by a future batch in this phase).
- Every Concept page in this batch contains `## What it is`, `## Why it exists`, `## Where it fits`, `## Key types`, `## See also`.
- Every Usage Guide contains `## Overview`, `## Minimal example`, `## Common patterns`, `## Pitfalls`, `## Reference cross-link`.
- Every Concept page's "See also" links to `../tutorials/governance-walkthrough.md`.
- `approvals` Usage Guide example mirrors `examples/approval_demo.py` enqueue/decide pattern (`list_pending` → `POST /deployments/{ref}/approvals/{id}/resolve`).
- `audit` Usage Guide shows `AuditRepository.list(AuditQuery(run_id=...))` + `GET /runs/{run_id}/timeline`.
- `policy` Usage Guide shows a `PolicyDefinition(denied_capabilities=[Capability.NETWORK_WRITE])` rule blocking a tool node, matching the scenario-3 wiring in `examples/governance_walkthrough.py`.
- Cross-links honored: `guardrails ↔ policy`, `identity ↔ approvals`/`service`, `approvals ↔ policy ↔ audit`.

## API Surprises / Drift Notes

1. **`ServiceRole.AUDITOR` does not exist.** `examples/approval_demo.py` and `examples/governance_walkthrough.py` both construct `StaticApiKeyCredential` with `ServiceRole.AUDITOR`, but `src/zeroth/core/identity/models.py` only defines `OPERATOR`, `REVIEWER`, `ADMIN`. The docs follow the source, not the example. Worth tracking as a separate example-file cleanup (out of scope for this plan; noted for the phase-level deferred list).
2. **`CapabilityRegistry` is not self-populating.** Every `Capability` value must be registered individually — the governance walkthrough does this in a `for cap in Capability:` loop; the `policy` Usage Guide preserves that pattern to avoid a footgun where node `capability_bindings` cannot be resolved.
3. **Policy denials do not raise.** A policy violation terminates the run with `RunStatus.FAILED` and `failure_state.reason == "policy_violation"`; callers of `run_graph` receive a returned `Run`, not an exception. The Pitfalls section of `how-to/policy.md` calls this out explicitly.
4. **Guardrail checks are wire-your-own.** `TokenBucketRateLimiter` and `QuotaEnforcer` are building blocks, not auto-wired middleware — the Usage Guide shows the caller explicitly invoking `check_and_consume` / `check_and_increment` because that is how the source is structured today.
5. **`deployments.md` concept missing.** Another batch in Phase 31 owns the `concepts/deployments.md` page; it was not present when this plan built, so one forward-reference link was rephrased to plain text to keep `mkdocs build` clean.

## Deviations from Plan

None — plan executed as written. All guardrails, identity, policy, approvals, and audit pages conform to the templates specified in the plan frontmatter.

## Ready For

Plan 31-04 (next governance-adjacent batch or cookbook, depending on wave ordering).

## Self-Check: PASSED

- All 10 files exist on disk.
- `git log --oneline --all | grep -E "7ec79bb|a40d594"` → both commits present.
- `mkdocs build` → 0 link warnings.
- Every concept page has the 5 required H2 sections and the governance-walkthrough tutorial link.
- `examples/approval_demo.py` enqueue/decide shape reproduced in `docs/how-to/approvals.md`.
