---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 04
subsystem: docs
tags: [docs, concepts, how-to, secrets, dispatch, econ, service, webhooks]
requires: []
provides:
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
affects: []
tech-stack:
  added: []
  patterns:
    - "Diataxis Concept + Usage Guide pairing"
    - "subsystem cross-linking via relative markdown paths"
key-files:
  created:
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
  modified: []
key-decisions:
  - "Substituted `webhooks` for `threads` as the 20th subsystem. `zeroth.core.threads` does not exist in the tree; `zeroth.core.webhooks` is a concrete user-facing async integration surface with no other documentation coverage, and is a better fit for the Concepts quadrant than `deployments` (which is operational and will be covered in Phase 32)."
  - "Wrote the service page around the real `entrypoint.app_factory` uvicorn factory, including the `python -m zeroth.core.service.entrypoint` path and the migration-on-boot behaviour in main()."
  - "Econ page explicitly cites `econ-instrumentation-sdk` and Regulus as the external companion, matching the `pyproject.toml` dependency name verbatim."
requirements-completed:
  - DOCS-03
  - DOCS-04
duration: "~25 min"
completed: 2026-04-11
---

# Phase 31 Plan 04: Subsystems Batch D (Platform) Summary

Ten Diataxis-paired documentation pages for the platform slice of
Zeroth: secrets, dispatch, economics, service, and webhooks — each with
one ~300-word Concept page and one ~400-500-word Usage Guide.

## Overview

- **Plan:** 31-04 Subsystems Batch D (Platform)
- **Duration:** ~25 min
- **Start:** 2026-04-11T20:18Z
- **End:** 2026-04-11T20:44Z
- **Tasks:** 2/2 complete
- **Files created:** 10

## Tasks

### Task 1 — Concept + Usage Guide for secrets, econ, service

Wrote six pages:

- `docs/concepts/secrets.md` — `SecretProvider`, `EnvSecretProvider`,
  `SecretResolver`, `SecretRedactor`; cross-links to service + identity.
- `docs/concepts/econ.md` — `InstrumentedProviderAdapter`, `RegulusClient`,
  `CostEstimator`, `BudgetEnforcer`; cites `econ-instrumentation-sdk` and
  Regulus as the external cost-observability companion.
- `docs/concepts/service.md` — `create_app`, `ServiceBootstrap`,
  `bootstrap_service`, `DeploymentBootstrapError`, `entrypoint.app_factory`.
- `docs/how-to/secrets.md` — minimal `EnvSecretProvider` + `SecretResolver`
  + `SecretRedactor` example; pitfalls cover rotation, scoping, plaintext
  env risk.
- `docs/how-to/econ.md` — wrapping a `ProviderAdapter` with
  `InstrumentedProviderAdapter`, pre-run `BudgetEnforcer` gate, patterns
  for budget caps and fail-open semantics.
- `docs/how-to/service.md` — real `uvicorn zeroth.core.service.entrypoint:app_factory --factory`
  command, `python -m zeroth.core.service.entrypoint` for Alembic-on-boot,
  and the documented startup ordering from `bootstrap.py`.

**Commit:** `2722c94`

### Task 2 — Concept + Usage Guide for dispatch and webhooks

Wrote four pages:

- `docs/concepts/dispatch.md` — `RunWorker`, `LeaseManager`, arq wakeup
  helpers; explicit note that Redis is a notification channel, not the
  authoritative queue.
- `docs/concepts/webhooks.md` — `WebhookService`, `WebhookSubscription`,
  `WebhookDelivery`, `WebhookDeadLetter`, `sign_payload`. Opens with the
  required threads-to-webhooks substitution note.
- `docs/how-to/dispatch.md` — `pip install 'zeroth-core[dispatch]'` /
  `uv add 'zeroth-core[dispatch]'`, `RunWorker` construction, optional
  `create_arq_pool` + `enqueue_wakeup` for low-latency dispatch.
- `docs/how-to/webhooks.md` — REST subscription example, `WebhookService.emit_event`
  from code, and a receiver-side HMAC-SHA256 verification snippet.

**Commit:** `ba6fa31`

## Threads → webhooks substitution

The Phase 31 content spec lists `threads` as subsystem #20 but also
grants the planner explicit discretion when `zeroth.core.threads` does
not exist. It does not exist in this tree. Rather than invent a page or
drop to 19 subsystems, the slot is filled with `zeroth.core.webhooks`:

- **It is a real, shipped subsystem.** `src/zeroth/core/webhooks/`
  exists with models, repository, signing, delivery worker, and service.
- **It is user-facing.** Webhook subscriptions are configured by tenants;
  delivery payloads cross the trust boundary.
- **It has no other coverage.** `deployments` is a closer structural
  match but belongs in the Phase 32 Deployment Guide; `webhooks` would
  otherwise be undocumented forever.
- **The substitution is auditable.** It is recorded in the plan
  frontmatter (`subsystem_substitution`), re-stated at the top of
  `docs/concepts/webhooks.md`, and re-stated here.

## Notes for plan 31-05 (cookbook + nav)

While reading source I captured details 31-05 will need:

- **Service entrypoint path:** `zeroth.core.service.entrypoint:app_factory`
  (NOT `zeroth.core.service.entrypoint:app`). It's a factory function;
  uvicorn must be invoked with `--factory` or the `factory=True` kwarg.
  `main()` also runs Alembic migrations against Postgres before booting.
- **Dispatch extra:** `zeroth-core[dispatch]` pulls `redis>=5.0.0` and
  `arq>=0.27`. ARQ wakeup is strictly a notification channel; the
  Postgres lease store in `zeroth.core.dispatch.lease` is authoritative.
- **Webhooks repository API:** `WebhookService.emit_event(event_type,
  deployment_ref, tenant_id, data) -> list[WebhookDelivery]` is the
  primary emitter; fan-out happens inside the service by
  `list_subscriptions_for_event`.
- **Econ dependency:** `econ-instrumentation-sdk>=0.1.1` is already a
  direct dependency in `pyproject.toml`, not an extra — cookbook recipes
  can assume it's installed.
- **Service app mounting:** `create_app` returns a `FastAPI` instance
  with a lifespan that starts `worker`, `queue_gauge`, `delivery_worker`,
  and `sla_checker_task`. Any cookbook recipe that constructs an app
  outside uvicorn must go through `bootstrap_service`.

## Verification

- `uv run mkdocs build` (non-strict) — **PASS**. The build surfaces
  pre-existing warnings about `concepts/identity.md` and
  `concepts/storage.md` not yet existing; these pages belong to batches
  B and C respectively and are outside the scope of batch D.
- All 10 files exist.
- All minimum line counts met (concepts ≥40 lines, how-to ≥50 lines).
- `docs/concepts/econ.md` contains the literal string "econ-instrumentation-sdk"
  and "Regulus".
- `docs/how-to/dispatch.md` contains `zeroth-core[dispatch]`.
- `docs/concepts/webhooks.md` contains the substitution note (matches
  `threads` + `substitut` patterns from the plan's verify block).
- `docs/how-to/service.md` contains both `entrypoint` and `uvicorn`.
- `mkdocs.yml` **was not modified** (plan 31-05 handles nav).

## Deviations from Plan

None — plan executed exactly as written.

## Authentication Gates

None.

## Issues Encountered

None.

## Next Phase Readiness

Ready for plan `31-05-cookbook-examples-and-nav-finalize`. With this
plan complete, all four subsystem batches (A graph/execution, B
data/state, C governance, D platform) have shipped their Concept +
Usage Guide pages. Plan 31-05 can now finalize `mkdocs.yml` navigation,
write the 10 cookbook recipes, add the matching runnable examples, and
extend `.github/workflows/examples.yml`.

## Self-Check: PASSED

- `docs/concepts/secrets.md` — FOUND
- `docs/concepts/dispatch.md` — FOUND
- `docs/concepts/econ.md` — FOUND
- `docs/concepts/service.md` — FOUND
- `docs/concepts/webhooks.md` — FOUND
- `docs/how-to/secrets.md` — FOUND
- `docs/how-to/dispatch.md` — FOUND
- `docs/how-to/econ.md` — FOUND
- `docs/how-to/service.md` — FOUND
- `docs/how-to/webhooks.md` — FOUND
- Commit `2722c94` (Task 1) — FOUND
- Commit `ba6fa31` (Task 2) — FOUND
