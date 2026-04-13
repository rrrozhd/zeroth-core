# Phase 41: Phase 40 Completion & Verification - Research

**Researched:** 2026-04-13
**Domain:** Verification, documentation updates, test regression gating
**Confidence:** HIGH

## Summary

Phase 41 is a verification and gap-closure phase that completes the work left unfinished when Phase 40 Plan 03 was not executed. The v4.0 milestone audit identified seven gaps: D-01 through D-05 are implemented but lack formal verification (no VERIFICATION.md), D-06 (full test regression gate) was never formally run through a plan, and D-07 (in-repo docs) was never updated. All five v4.0 subsystems are implemented and tested (1199 tests pass, 0 failures), but the formal verification artifacts and documentation updates are missing.

This phase has three concrete workstreams: (1) run the full test suite and capture formal regression evidence, (2) update in-repo documentation (concept stubs and HTTP API reference) to describe v4.0 API capabilities, and (3) create a Phase 40 VERIFICATION.md that maps D-01 through D-07 to concrete evidence. Additionally, the `docs/assets/openapi/` copy of the OpenAPI spec is stale and must be synced -- it is missing the artifact and template endpoint definitions added in Plan 40-02.

**Primary recommendation:** Execute the unfinished 40-03 plan work (regression gate + docs updates), then produce VERIFICATION.md as a formal artifact linking all seven D-requirements to evidence.

## Project Constraints (from CLAUDE.md)

- **Build/test commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/` [VERIFIED: CLAUDE.md]
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill [VERIFIED: CLAUDE.md]
- **Implementation tracking:** PROGRESS.md is single source of truth; phases use `phases/phase-N-*/PLAN.md` and `phases/phase-N-*/artifacts/` [VERIFIED: CLAUDE.md]
- **Context efficiency:** Read only what is needed for the task [VERIFIED: CLAUDE.md]

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
None -- infrastructure phase, all decisions at Claude's discretion.

### Claude's Discretion
All implementation choices are at Claude's discretion -- pure infrastructure/verification phase. Use ROADMAP phase goal, success criteria, and codebase conventions to guide decisions.

### Deferred Ideas (OUT OF SCOPE)
None -- infrastructure phase.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| D-01 | All v4.0 subsystems on ServiceBootstrap after bootstrap_service() | Plans 40-01 completed: 5 bootstrap validation tests in `tests/test_v4_bootstrap_validation.py` confirm `artifact_store`, `http_client`, `template_registry`, `subgraph_executor`, `context_window_enabled` all non-None. Needs VERIFICATION.md linkage. |
| D-02 | Cross-feature interactions tested | Plan 40-01 completed: 6 cross-feature integration tests in `tests/test_v4_cross_feature_integration.py`. Needs VERIFICATION.md linkage. |
| D-03 | Artifact retrieval REST endpoint (GET /v1/artifacts/{id}) | Plan 40-02 completed: `src/zeroth/core/service/artifact_api.py` with 6 tests in `tests/service/test_artifact_api.py`. Needs VERIFICATION.md linkage. |
| D-04 | Template CRUD REST endpoints (GET/POST/DELETE /v1/templates) | Plan 40-02 completed: `src/zeroth/core/service/template_api.py` with 6 tests in `tests/service/test_template_api.py`. Design debt noted (DELETE accesses private `_templates` dict). Needs VERIFICATION.md linkage. |
| D-05 | SubgraphNode-in-parallel rejected with clear validation error | Plan 40-01 completed: Guard in `src/zeroth/core/parallel/executor.py` lines 67-70. Tested in both bootstrap validation and cross-feature tests. Needs VERIFICATION.md linkage. |
| D-06 | Full test suite passes with zero new failures (backward compatibility) | NOT YET DONE. Integration checker confirmed 1199 passed / 0 failed, but this was not formally gated through plan execution. Phase 41 must run `uv run pytest -v` and capture output as a formal artifact. |
| D-07 | In-repo documentation references new v4.0 API capabilities | NOT YET DONE. Concept stub pages exist (artifacts.md, templates.md, parallel.md, subgraph.md) but contain only boilerplate "See API Reference" text. HTTP API reference page (`docs/reference/http-api.md`) references OpenAPI spec via Swagger UI, but `docs/assets/openapi/zeroth-core-openapi.json` is STALE (missing v4.0 endpoints). |
</phase_requirements>

## Standard Stack

No new libraries needed. This phase uses only existing project infrastructure.

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pytest | (existing) | Test suite execution and regression gating | Project standard per CLAUDE.md |
| ruff | (existing) | Lint and format verification | Project standard per CLAUDE.md |
| mkdocs-material | (existing) | Documentation build verification | Already configured in project |

[VERIFIED: CLAUDE.md, pyproject.toml]

## Architecture Patterns

### Phase 40 VERIFICATION.md Structure

The established verification format from phases 35-39 uses YAML frontmatter with phase metadata, followed by sections for Observable Truths, Required Artifacts, Key Link Verification, Behavioral Spot-Checks, Requirements Coverage, and Gaps Summary. [VERIFIED: reviewed 38-VERIFICATION.md and 39-VERIFICATION.md]

```yaml
---
phase: 40-integration-service-wiring
verified: {ISO timestamp}
status: passed
score: N/N must-haves verified
overrides_applied: 0
---
```

### Key Sections Required

1. **Observable Truths table** -- one row per must-have from plans 40-01 and 40-02 frontmatter
2. **Required Artifacts table** -- all files created/modified by plans 40-01 and 40-02
3. **Key Link Verification** -- cross-module wiring (bootstrap -> API routes -> subsystems)
4. **Behavioral Spot-Checks** -- actual test commands and outputs
5. **Requirements Coverage** -- D-01 through D-07 mapped to evidence
6. **Gaps Summary** -- declare gaps closed or remaining

[VERIFIED: 38-VERIFICATION.md, 39-VERIFICATION.md format analysis]

### Documentation Update Pattern

The v4.0 concept pages are currently stubs (3-5 lines each). They need substantive content describing:
- What the feature does
- Key API surface (REST endpoints for artifacts and templates)
- Known limitations (SubgraphNode-in-parallel for parallel.md)

The `docs/reference/http-api.md` page renders the OpenAPI spec via Swagger UI from `docs/assets/openapi/zeroth-core-openapi.json`. This file is currently STALE -- missing the v4.0 artifact and template endpoints. It must be synced from `openapi/zeroth-core-openapi.json`. [VERIFIED: diff comparison showed docs copy missing CreateTemplateRequest, TemplateListResponse, TemplateResponse schemas and /v1/artifacts/*, /v1/templates/* paths]

The README.md currently has no mention of v4.0 capabilities (parallel, subgraph, templates, artifacts, context window, HTTP client). A brief addition to the "Key Concepts" or "Architecture Overview" section would satisfy D-07. [VERIFIED: grep found zero matches for artifact/template/SubgraphNode/v4.0/parallel in README.md]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Test regression evidence | Manual test counting | `uv run pytest -v --tb=short 2>&1 \| tee` | Captures exact output, machine-parseable |
| OpenAPI spec sync | Manual copy/edit | `cp openapi/zeroth-core-openapi.json docs/assets/openapi/` | Canonical spec is generated by `scripts/dump_openapi.py` |
| VERIFICATION.md | Free-form prose | Established table-driven format from phases 35-39 | Consistency with prior verification reports |

## Common Pitfalls

### Pitfall 1: Stale OpenAPI in docs/assets
**What goes wrong:** The `docs/assets/openapi/zeroth-core-openapi.json` is a copy of the canonical spec but was not updated after Plan 40-02 added artifact and template endpoints. The Swagger UI on the HTTP API reference page will not show v4.0 endpoints until this file is synced.
**Why it happens:** Plan 40-02 regenerated `openapi/zeroth-core-openapi.json` but the copy step to `docs/assets/` was omitted.
**How to avoid:** Copy the canonical spec to docs/assets as part of the documentation update task.
**Warning signs:** Swagger UI on the docs site missing artifact/template endpoints.

[VERIFIED: diff between openapi/zeroth-core-openapi.json and docs/assets/openapi/zeroth-core-openapi.json shows the docs copy is missing all v4.0 endpoint definitions]

### Pitfall 2: Concept stubs are too thin for D-07
**What goes wrong:** The current concept pages (artifacts.md, templates.md, parallel.md, subgraph.md, context-window.md, http-client.md) each contain only 3-5 lines of boilerplate. Technically they "reference" the API but provide no useful information about v4.0 capabilities.
**Why it happens:** Plan 40-03 created minimal stubs as placeholders but was never executed to flesh them out.
**How to avoid:** Add at least: (a) a brief description of the feature, (b) key configuration or API surface, (c) known limitations where applicable. Keep it concise -- these are concept pages, not full guides.

[VERIFIED: read all six v4.0 concept stub pages -- each is 3 lines: title, "Added in v4.0", and a link to API reference]

### Pitfall 3: Phase 40 has no artifacts directory
**What goes wrong:** Test regression output cannot be saved as a formal artifact.
**Why it happens:** Plans 40-01 and 40-02 did not create the artifacts directory.
**How to avoid:** Create `phases/40-integration-service-wiring/artifacts/` before saving test output.

[VERIFIED: ls confirmed no artifacts directory exists under phase 40]

### Pitfall 4: README.md does not mention v4.0 at all
**What goes wrong:** D-07 requires "in-repo documentation references new v4.0 API capabilities." The README is the most visible in-repo document but has no mention of any v4.0 feature (parallel, subgraph, templates, artifacts, context window, HTTP client).
**Why it happens:** v4.0 work focused on runtime implementation; README was last updated during v3.0.
**How to avoid:** Add a brief section or update the existing "Key Concepts" section to mention v4.0 capabilities. Keep it proportional -- a few sentences or a feature list, not a full guide.

[VERIFIED: grep found zero matches for v4.0 feature terms in README.md]

### Pitfall 5: PROGRESS.md missing Phase 40 entries
**What goes wrong:** Phase 40 has no entries in PROGRESS.md despite plans 40-01 and 40-02 being complete.
**Why it happens:** Progress logging may have been missed during rapid Phase 40 execution.
**How to avoid:** This is Phase 42 scope (milestone hygiene), not Phase 41. Note it but don't block on it.

[VERIFIED: grep found no Phase 40 references in PROGRESS.md]

## Code Examples

### Running the regression gate
```bash
# Source: CLAUDE.md build & test section, Plan 40-03 Task 1
uv run pytest -v --tb=short 2>&1 | tee /tmp/phase40-full-regression.txt
```

Expected output: `1199 passed, 12 deselected` (or more, if tests were added since last count).

### Syncing OpenAPI spec to docs
```bash
# Source: docs/reference/http-api.md "Regenerating the spec" section
cp openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json
```

### Lint verification
```bash
# Source: CLAUDE.md build & test section
uv run ruff check src/
uv run ruff format --check src/
```

## Existing Evidence Inventory

Evidence already available from Phase 40 plan execution (40-01 and 40-02 summaries):

| Requirement | Evidence Location | Status |
|-------------|-------------------|--------|
| D-01 | `tests/test_v4_bootstrap_validation.py` (5 tests) | Implemented, needs verification artifact |
| D-02 | `tests/test_v4_cross_feature_integration.py` (6 tests) | Implemented, needs verification artifact |
| D-03 | `src/zeroth/core/service/artifact_api.py`, `tests/service/test_artifact_api.py` (6 tests) | Implemented, needs verification artifact |
| D-04 | `src/zeroth/core/service/template_api.py`, `tests/service/test_template_api.py` (6 tests) | Implemented, needs verification artifact |
| D-05 | `src/zeroth/core/parallel/executor.py` lines 67-70, tested in both validation suites | Implemented, needs verification artifact |
| D-06 | Integration checker ran full suite (1199/0), but no formal plan-gated artifact | NOT FORMALLY GATED -- must run and capture |
| D-07 | Concept stubs exist but are boilerplate; docs/assets OpenAPI is stale; README has no v4.0 mention | NOT DONE -- must update docs |

[VERIFIED: read all 40-01-SUMMARY.md, 40-02-SUMMARY.md, milestone audit, and actual source files]

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Plan 40-03 as part of Phase 40 | Moved to Phase 41 as gap closure | 2026-04-13 (v4.0 audit) | Same work, different phase boundary |
| Concept stubs as placeholders | Need substantive content per D-07 | Phase 41 | Stubs must be fleshed out |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | mkdocs build will pass after doc updates (mkdocs-material is configured and working) | Pitfall 2 | LOW -- docs site was built successfully in Phases 30-32, but config may have drifted |

## Open Questions

1. **How much content should concept stubs contain?**
   - What we know: Current stubs are 3 lines each. D-07 says "references v4.0 API capabilities."
   - What's unclear: Whether a brief paragraph + endpoint list is sufficient, or full concept pages are expected.
   - Recommendation: Add a meaningful paragraph describing the feature, list the REST endpoints (for artifacts/templates), note the SubgraphNode-in-parallel limitation, and link to API reference. Keep it concise -- full concept guides are future work (phases 30-32 already covered the in-depth docs pattern for v3.0 subsystems).

2. **Should README.md be updated?**
   - What we know: README has zero v4.0 mentions. D-07 says "in-repo documentation references new v4.0 API capabilities."
   - What's unclear: Whether "in-repo documentation" means just the docs site content or also the README.
   - Recommendation: Add a brief v4.0 section to README since it's the most visible in-repo document. A "v4.0 Platform Extensions" subsection under Key Concepts listing the six new subsystems would be appropriate and lightweight.

## Sources

### Primary (HIGH confidence)
- `.planning/v4.0-MILESTONE-AUDIT.md` -- gap analysis identifying all seven D-requirement gaps
- `.planning/phases/40-integration-service-wiring/40-01-SUMMARY.md` -- Plan 01 completion evidence
- `.planning/phases/40-integration-service-wiring/40-02-SUMMARY.md` -- Plan 02 completion evidence
- `.planning/phases/40-integration-service-wiring/40-03-PLAN.md` -- Unexecuted Plan 03 (regression + docs)
- `.planning/phases/38-parallel-fan-out-fan-in/38-VERIFICATION.md` -- VERIFICATION.md format reference
- `.planning/phases/39-subgraph-composition/39-VERIFICATION.md` -- VERIFICATION.md format reference
- `docs/concepts/*.md` -- current state of concept stub pages
- `docs/reference/http-api.md` -- current HTTP API reference page
- `openapi/zeroth-core-openapi.json` vs `docs/assets/openapi/zeroth-core-openapi.json` -- diff confirming stale copy
- `README.md` -- current state, no v4.0 content
- `uv run pytest --collect-only -q` -- confirmed 1199 tests collected, 12 deselected

### Secondary (MEDIUM confidence)
- None

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new libraries, existing infrastructure only
- Architecture: HIGH -- VERIFICATION.md format well-established from 5 prior phases
- Pitfalls: HIGH -- all verified by direct file reads and diffs

**Research date:** 2026-04-13
**Valid until:** 2026-04-27 (14 days -- stable, no external dependencies)
