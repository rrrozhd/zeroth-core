# Zeroth project model

## Purpose and readiness boundary

Zeroth is a governed, durable multi-agent runtime. Production readiness requires
all release gates to pass for one exact candidate, independent acceptance of any
mathematical authorization claim, and deployed acceptance against a named target
with its real configuration, SLOs, external systems, provenance, rollback, and
human promotion signoff. Local tests or a green subset of CI do not establish
that broader claim.

## Runtime and critical flows

- `src/zeroth/service/app.py` creates the FastAPI service from a scoped
  bootstrap. Production configuration must supply durable storage, shared
  browser-session signing, and the enabled external systems.
- `src/zeroth/service/api/run_api.py` validates identity, scope, contracts, and
  guardrails before persisting a run. Guarded admission serializes rate, quota,
  queue, and worker-capacity decisions by tenant/workspace/deployment.
- `src/zeroth/integrations/persistence/runs/run_repository.py` commits run,
  thread, and checkpoint state atomically. Thread and admission coordination
  rows are locked before read-modify-write updates.
- `src/zeroth/runtime/orchestration/run_worker.py` claims durable work through
  `src/zeroth/platform/dispatch/lease.py`, installs a generation fence, renews
  the lease, drives the graph, and refuses stale writes after displacement.
- `src/zeroth/platform/storage/async_postgres.py` owns PostgreSQL transaction
  lifecycle. Transaction cleanup must finish before a cancelled connection is
  returned to the pool.
- Audit, certification, retention, repository ingress, untrusted execution,
  LangGraph compatibility, and economic evidence are separate release-gate
  surfaces. Their passing tests are scoped evidence, not interchangeable proof.

## Invariants

1. Tenant and workspace scope is structurally bound to every persistence and
   execution operation; missing or conflicting ownership fails closed.
2. A run has at most one live lease generation. State writes require the live
   worker and generation, and accepted runs may be neither lost nor duplicated.
3. Run, thread, and checkpoint changes commit or roll back together. Concurrent
   writers lock the thread before deriving checkpoint order or merging refs.
4. Guardrail admission and worker capacity use the same scoped coordination
   lock. Optimizations may remove redundant warm-row writes but may not weaken
   the atomic decision or its saturation evidence.
5. Candidate evidence is valid only for its measured commit, package/image
   digests, configuration, service instances, and runner capacity.
6. Statistical baselines require replicated isolated samples. One noisy hosted
   runner result is insufficient to justify changing code or thresholds.
7. Mathematical development data cannot confirm a selected construction.
   Rejected constructions authorize nothing, and confirmation roots remain
   unspent until a useful candidate and independent review exist.

## Major decisions

- Python authentication uses PyJWT rather than `python-jose`; locked and fresh
  downstream wheel audits currently report no known Python vulnerabilities.
- Runtime installation uses hashed dependency floors and removes package
  installers from the final image.
- Load diagnostics are excluded from timed release measurements because their
  monkeypatching perturbed the measured hot path. Explicit diagnostic runs remain
  available for causal investigation.
- The stored load thresholds remain fixed. They must not be raised to fit a
  failing candidate.
- An advisory worker pending-query optimization was reverted after an exact ARM
  run worsened overload latency. Warm admission/thread lock-first changes retain
  the original atomic paths while removing conflict writes.
- Both tested mathematical constructions were rejected by their preregistered
  useful-power criterion. Public authorization remains closed.

## Verification and debugging

- The executable gate contract is `release/gates/release-gates.json`; workflow
  wiring is in `.github/workflows/release-gates.yml` and release workflows.
- Load profiles and fixed thresholds are in `release/load/profiles-v1.json`,
  `release/load/baseline-v1.json`, and `release/load/report.py`. Preserve raw
  request artifacts and compare distributions across isolated exact-SHA runs.
- Start dispatch debugging at run status, lease worker/generation/expiry,
  checkpoint refs, and worker/audit events. PostgreSQL `INTRANS` pool-discard
  warnings indicate interrupted transaction cleanup and are release blockers.
- Run the full repository suite before promotion. For persistence or dispatch
  changes, also run the focused storage, dispatch, persistence, and load suites.
- Security evidence consists of the exact candidate matrix, dependency audits,
  image scan, SBOM, provenance, and deployed boundary checks. No one layer
  substitutes for another.

## Deployment and rollback

The normal release path builds once, gathers candidate-bound gates, publishes
to TestPyPI only after candidate validation, performs remote economic acceptance,
and requires a named human plus protected `pypi` environment for promotion.
A proper production deployment additionally needs a named target, immutable
image digest, target configuration and secrets, migrations, health and SLO
checks, external-system verification, drain/restart evidence, and a tested
rollback to the prior immutable image and compatible schema.

No production target is currently named in repository evidence. Do not fabricate
deployment, certification, signature, or promotion records. Runtime changes on
the readiness branch can be rolled back by reverting their individual commits;
no schema migration is introduced by the current load or transaction fixes.

## Current unresolved work

- The exact ARM load gate still exceeds the fixed burst/overload latency envelope
  and has reproduced a stranded accepted run during cancellation cleanup.
- The runtime image retains Debian scanner findings for which the pinned
  Bookworm repositories currently report no fixes; the separately tested PCRE2
  update removes one finding but does not clear the critical/high set.
- The two mathematical constructions fail useful power and have no independent
  acceptance or production integration path.
- A concrete production target, latency SLO, credentials/configuration,
  candidate release, deployed acceptance, rollback exercise, and named human
  promotion signoff are absent.
