# Phase 27: Monolith Archive & Namespace Rename - Research

**Researched:** 2026-04-10
**Domain:** Git archive preservation, namespace-package migration, import-path codemods, verification gates
**Confidence:** HIGH

## Summary

Phase 27 is a structural migration, not a feature phase. The repo must be archived before any rename work starts, then the Python package tree must move from `zeroth.*` to `zeroth.core.*` without changing behavior. The safest implementation is a four-step sequence: inventory and baseline the current repo, create reproducible archive scripts plus evidence, perform a `git mv` relocation plus scripted path rewrites, then add docstring/verification gates and rerun the full suite against the renamed layout.

The current repo state materially affects the plan. There are 36 worktrees, 84 branches, 2 stashes, and 2 detached-HEAD worktrees right now. The Phase 27 context assumed a single detached-HEAD worktree, so the execution plan must enumerate detached worktrees dynamically instead of hard-coding a single branch. The source tree currently has 27 first-level Python subpackages under `src/zeroth/`, `pyproject.toml` still ships `name = "zeroth"` with `packages = ["src/zeroth"]`, `Dockerfile` still launches `python -m zeroth.service.entrypoint`, and there is no existing `scripts/` directory or `.github/workflows/` CI to extend. Phase 27 therefore needs to create its own migration scripts and its own first CI workflow for the docstring gate.

**Primary recommendation:** split execution into four plans:
1. Archive preflight, local tarball/mirror creation, and baseline pytest evidence.
2. GitHub archive publication and archive notice.
3. Package relocation, import rewrites, and packaging/path fixes.
4. Docstring coverage, CI gate, and post-rename verification.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Perform the rename in-place in this repo with `git mv`; do not copy `/tmp/zeroth-split/zeroth-core-build/`.
- D-02: Move every current Python subpackage under `src/zeroth/core/`, create `src/zeroth/core/__init__.py`, and delete `src/zeroth/__init__.py`.
- D-03: Rewrite Python imports with `libcst`, covering direct imports and `importlib.import_module("zeroth.X")`.
- D-04: Rewrite non-Python references with a scripted, explicit pass; commit the migration script under `scripts/rename_to_zeroth_core.py`.
- D-07/D-08: Keep work atomic and reviewable; the move + import rewrite + top-level `__init__.py` removal must land in one coherent refactor step so the tree never sits half-renamed.
- D-09/D-10/D-12/D-13: Rename packaging metadata to `zeroth-core`, point the wheel target at `src/zeroth/core`, and update any console-script or module-path strings.
- D-14 through D-19: Add `interrogate` + Google-style docstring enforcement with a hard fail-under of 90% on the public `zeroth.core.*` surface.
- D-20 through D-23: Capture pre-rename and post-rename pytest logs and compare them; no new skips are allowed.
- D-24 through D-30: Archive first, then rename; preserve stashes and detached worktrees as named refs, create tarball + bare mirror + GitHub mirror, and prove recoverability.
- D-31 through D-33: Enforce PEP 420 by removing `src/zeroth/__init__.py`, making `zeroth` a namespace package, and ensuring `src/zeroth/` contains only `core/` afterward.

### Repo Findings That Change Execution Detail
- `git worktree list | wc -l` is `36`.
- `git worktree list | rg -c '\\(detached HEAD\\)'` is `2`, not `1`.
- `git branch -a | wc -l` is `84`.
- `git stash list` shows `2` stashes.
- `find src/zeroth -maxdepth 1 -mindepth 1 -type d | rg -v '__pycache__' | wc -l` is `27`.
- There is no `scripts/` directory yet, so archive and rename automation must be created from scratch.
- There is no `.github/workflows/` directory yet, so the interrogate CI gate must be introduced rather than updated.

### Deferred / Out of Scope
- Publishing `econ-instrumentation-sdk` or `zeroth-core` to PyPI.
- Moving Studio into its own repo.
- Docs site scaffolding or content beyond docstrings needed for API reference readiness.
- Any runtime behavior change.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ARCHIVE-01 | Multi-layer monolith archive exists and is documented | Archive scripts, artifact logs, local tarball, local mirror, GitHub mirror publication |
| ARCHIVE-02 | All worktrees/stashes/detached HEADs are preserved and recoverable | Dynamic inventory + synthesized `archive/*` refs + recovery-test script |
| ARCHIVE-03 | Archive repo carries a visible archived notice | GitHub publication step must prepend README banner and set repo description/archive flag |
| RENAME-01 | All Python source moves to `zeroth.core.*` with zero functional change | `git mv` of all 27 subpackages plus tree-wide import rewrite |
| RENAME-02 | `zeroth` becomes a PEP 420 namespace package | Drop `src/zeroth/__init__.py`, add `src/zeroth/core/__init__.py`, verify namespace semantics |
| RENAME-03 | All imports, entry points, and strings point at `zeroth.core.*` | `libcst` Python rewrites + non-Python path rewrite pass |
| RENAME-04 | Existing test suite passes with no new skips/regressions | Baseline pytest log before rename + post-rename log diff + import smoke tests |
| RENAME-05 | Public surface reaches >=90% Google-style docstring coverage | `interrogate` + Ruff pydocstyle configuration, docstring gap-filling, CI enforcement |
</phase_requirements>

## Current Repo Findings

### Packaging and Runtime Entry Points
- `pyproject.toml` currently declares `name = "zeroth"` and wheel packages `["src/zeroth"]`.
- `Dockerfile` still uses `CMD ["python", "-m", "zeroth.service.entrypoint"]`.
- `alembic.ini` uses `script_location = src/zeroth/migrations`.
- Tests and source contain widespread `from zeroth...` imports, so the codemod must touch both `src/` and `tests/`.

### Source Layout
Current first-level packages under `src/zeroth/`:
- `agent_runtime`
- `approvals`
- `audit`
- `conditions`
- `config`
- `contracts`
- `demos`
- `deployments`
- `dispatch`
- `econ`
- `execution_units`
- `graph`
- `guardrails`
- `identity`
- `mappings`
- `memory`
- `migrations`
- `observability`
- `orchestrator`
- `policy`
- `runs`
- `sandbox_sidecar`
- `secrets`
- `service`
- `storage`
- `studio`
- `webhooks`

### Test and Tooling Reality
- There are 98 `test_*.py` files. The actual collected-test count must be measured with baseline pytest output instead of guessed from file count.
- No CI workflow exists today, so Phase 27 can legitimately add `.github/workflows/ci.yml` as the first repo-wide verification workflow.
- No migration helper scripts exist today, so reproducibility should be introduced via committed scripts rather than shell history.

## Standard Stack

### Tools Already Present
| Tool | Status | Use in Phase 27 |
|------|--------|-----------------|
| `uv` | present | dependency sync, pytest, Ruff, interrogate |
| `pytest` | in `dependency-groups.dev` | baseline and post-rename verification |
| `ruff` | in `dependency-groups.dev` | lint + Google-style pydocstyle enforcement |
| `hatchling` | build backend | namespace-package wheel config update |
| `git` | present | worktree/stash preservation, mirror creation, `git mv` |
| `gh` | assumed available for archive repo publishing | GitHub repo creation/edit/archive and mirror push |

### New Dev Dependencies Recommended
| Dependency | Why |
|------------|-----|
| `libcst` | required for format-preserving Python import rewrites |
| `interrogate` | required for docstring coverage measurement/gating |

## Architecture Patterns

### Pattern 1: Dynamic Detached-HEAD Preservation
Do not assume one detached worktree. Enumerate every `git worktree list` line containing `(detached HEAD)`, capture the SHA, and create a named branch such as `archive/detached-wt-<shortsha>` for each before cloning the mirror.

### Pattern 2: Two-Stage Rename Automation
Keep Python rewrites and non-Python rewrites separate:
- `scripts/rename_to_zeroth_core.py` uses `libcst` for `.py` files.
- A second explicit shell or Python pass rewrites `pyproject.toml`, `Dockerfile`, `docker-compose.yml`, `alembic.ini`, markdown, and any other text files containing `zeroth.` module paths.

### Pattern 3: Baseline-Then-Diff Verification
Capture evidence before changing the tree:
- `pytest-before-rename.log`
- archive inventory log
- worktree/stash preservation log

Then capture the same evidence after the rename:
- `pytest-after-rename.log`
- namespace smoke-test log
- interrogate coverage log

### Pattern 4: CI Introduction Instead Of CI Extension
Because `.github/workflows/` does not exist yet, the docstring gate should create a minimal CI workflow that runs:
- `uv sync --all-groups`
- `uv run ruff check src tests`
- `uv run pytest -v --no-header -ra`
- `uv run interrogate src/zeroth/core`

## Risks and Mitigations

### Risk 1: Archive plan is stale relative to live repo state
Mitigation: first plan must inventory branches, stashes, worktrees, and detached worktrees dynamically and write those values into an artifact before doing any archive operation.

### Risk 2: `git mv` plus import rewrite leaves the repo transiently broken
Mitigation: stage the move, rewrite imports immediately, update wheel metadata and path strings in the same execution plan, and gate on smoke-import commands before any verification run.

### Risk 3: Namespace-package config is correct for imports but wrong for wheel building
Mitigation: verify both runtime imports and built artifact metadata; the plan must not stop at successful `pytest` alone.

### Risk 4: Docstring coverage work balloons uncontrollably
Mitigation: measure interrogate baseline first, prioritize `__init__.py` exports and public service/orchestrator/graph/execution-unit surfaces, and stop once the measured threshold is met with real docstrings.

## Recommended Plan Split

### Plan 27-01
Archive preflight, synthesized archive refs, local tarball/bare mirror, and baseline pytest evidence.

### Plan 27-02
GitHub `rrrozhd/zeroth-archive` publication, README banner/description/archive flag, and remote recoverability evidence.

### Plan 27-03
`git mv` relocation into `src/zeroth/core/`, `libcst` import codemod, non-Python path rewrites, and PEP 420 packaging changes.

### Plan 27-04
Docstring tooling, CI gate, docstring gap-filling, post-rename verification, and baseline-vs-post-rename comparison artifacts.
