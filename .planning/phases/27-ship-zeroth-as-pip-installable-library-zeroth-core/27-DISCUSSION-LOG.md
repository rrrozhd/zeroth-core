# Phase 27: Monolith Archive & Namespace Rename - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in `27-CONTEXT.md` — this log preserves the alternatives considered and records the auto-selected choices in `--auto` mode.

**Date:** 2026-04-10
**Phase:** 27-ship-zeroth-as-pip-installable-library-zeroth-core
**Mode:** `/gsd:discuss-phase 27 --auto` — gray areas auto-selected, recommended defaults chosen for every decision
**Areas discussed:** Rename execution strategy, Commit granularity, Packaging metadata, Docstring coverage, Test suite verification, Archive mechanics, PEP 420 enforcement

---

## Rename Execution Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Adopt `/tmp/zeroth-split/zeroth-core-build/` wholesale | Copy the hand-executed rename tree over the current repo | |
| Rebuild in-place via `git mv` + scripted import rewrite (Recommended) | Preserves git history, uses libcst for AST-accurate import rewrites | ✓ |
| Selective merge | Cherry-pick pieces from `/tmp` tree; rebuild the rest | |

**Auto-selected:** Rebuild in-place via `git mv` + scripted import rewrite (Recommended)
**Why recommended:** Git history + blame preservation on every moved file is a non-negotiable; wholesale copy drops history. AST-level rewrites via libcst avoid false positives that regex-based sed produces on f-strings, triple-quoted code samples in docstrings, and string literals that happen to look like import paths. The `/tmp` tree is still valuable as a cross-check reference for the final layout.

---

## Commit Granularity

| Option | Description | Selected |
|--------|-------------|----------|
| Single mega-commit | One "refactor: rename to zeroth.core" commit containing everything | |
| Multiple atomic commits, one per logical step (Recommended) | Archive → file moves → import rewrite → non-Python refs → pyproject → tests → docstrings | ✓ |
| Per-subpackage commits | One commit per renamed subdirectory (27 commits for the move alone) | |

**Auto-selected:** Multiple atomic commits, one per logical step (Recommended)
**Why recommended:** Each commit is individually reviewable, git bisect stays useful, and failures are localized. The file-move + import-rewrite + __init__.py drop are combined into a single commit so every commit on `main` leaves the tree importable (the halfway state after `git mv` but before import rewrite would break every test).

---

## Packaging Metadata (Phase 27 vs Phase 28 split)

| Option | Description | Selected |
|--------|-------------|----------|
| Keep `name = "zeroth"`, defer all pyproject work to Phase 28 | Minimal touch | |
| Rename to `zeroth-core` + update wheel target now, publishing still in Phase 28 (Recommended) | Makes local `uv pip install -e .` work end-to-end with the new layout | ✓ |
| Rename AND publish to TestPyPI in this phase | Eager publishing | |

**Auto-selected:** Rename to `zeroth-core` + update wheel target now, publishing still in Phase 28 (Recommended)
**Why recommended:** The rename is not actually verifiable without the packaging metadata pointing at `src/zeroth/core`. Editable installs would fail and the test suite would not resolve imports. Publishing (PKG-02, PyPI trusted publisher, optional extras) is the separate Phase 28 deliverable and is not pre-empted here.

---

## Docstring Coverage Strategy (RENAME-05 ≥90% Google)

| Option | Description | Selected |
|--------|-------------|----------|
| Human-authored docstrings per subpackage, interrogate gate in CI (Recommended) | Quality docstrings, baseline-first, interrogate fail-under=90, ruff pydocstyle Google convention | ✓ |
| LLM auto-generate docstrings for everything | Fastest, lowest quality | |
| Defer docstring push to a later phase | Violates RENAME-05 acceptance criterion | |

**Auto-selected:** Human-authored docstrings per subpackage, interrogate gate in CI (Recommended)
**Why recommended:** The public API surface is what external users will read — placeholder "TODO: do the thing" docstrings pass interrogate but fail the actual intent of the requirement. Baseline-first (measure before writing) lets us size the work correctly and avoid rewriting already-good docstrings.

**Follow-up auto-decisions (per D-14 through D-19):**
- Tool: `interrogate` (locked by requirement hint)
- Style: Google (locked by RENAME-05 text)
- Threshold: fail-under = 90 (locked by RENAME-05 text)
- Enforcement tool: `ruff` D-rules with `convention = "google"` (recommended — ruff already in stack)
- Exclusions: migrations, tests, scripts, apps

---

## Test Suite Verification (RENAME-04)

| Option | Description | Selected |
|--------|-------------|----------|
| Run full suite before + after rename, diff results, grandfather pre-existing skips (Recommended) | Baseline-captured evidence; zero-new-skip rule | ✓ |
| Strict literal zero-skips interpretation | Eliminate pre-existing skips as well; may bloat scope | |
| Spot-check a subset of tests | Faster, less evidence | |

**Auto-selected:** Run full suite before + after rename, diff results, grandfather pre-existing skips (Recommended)
**Why recommended:** "Zero regressions" is the true intent of RENAME-04. Pre-existing skips that require Docker/Postgres/Redis services are environmental, not caused by the rename, and removing them is orthogonal work. The rule the executor follows is "no NEW skips introduced by the rename."

---

## Archive Mechanics (ARCHIVE-01/02/03)

| Option | Description | Selected |
|--------|-------------|----------|
| Tarball + `clone --mirror` + GitHub push, in that order, archive runs BEFORE any rename work (Recommended) | Captures pristine monolith state; three independent layers | ✓ |
| Archive after rename | Risks capturing already-modified state | |
| Tarball only (skip bare mirror + GitHub) | Fails ARCHIVE-01 (three layers required) | |

**Auto-selected:** Tarball + `clone --mirror` + GitHub push, in that order, BEFORE any rename work (Recommended)
**Why recommended:** The whole point of the archive is to preserve the pre-split monolith state. Any work done before the archive lives runs the risk of contaminating the snapshot. Three independent layers means any single layer can be lost without losing history.

### Stash + Detached-HEAD Worktree Preservation

| Option | Description | Selected |
|--------|-------------|----------|
| Convert stashes and detached worktrees to named `archive/*` branches before cloning mirror (Recommended) | Ensures `clone --mirror` captures them as real refs | ✓ |
| Trust `clone --mirror` to capture stashes automatically | Fails — stashes are not regular refs and detached worktrees are local-only | |

**Auto-selected:** Convert stashes and detached worktrees to named `archive/*` branches before cloning mirror (Recommended)
**Why recommended:** Validated empirically — `clone --mirror` only captures refs under `refs/`. Stashes live at `refs/stash` (singular) and a detached-HEAD worktree has no ref at all. Creating `refs/heads/archive/stash-0`, `refs/heads/archive/stash-1`, and `refs/heads/archive/detached-wt-<sha>` makes them visible to the mirror clone.

### GitHub Archive Repo Creation

| Option | Description | Selected |
|--------|-------------|----------|
| USER-ACTION checkpoint: user either confirms `gh repo create` invocation or creates the repo manually (Recommended) | Repo creation is a user-permission-gated action | ✓ |
| Claude auto-creates the GitHub repo without asking | Not allowed — repo/account creation needs explicit user authorization | |
| Skip GitHub layer | Fails ARCHIVE-01 | |

**Auto-selected:** USER-ACTION checkpoint before invoking `gh repo create` (Recommended)
**Why recommended:** Creating a GitHub repo expands the audience of the content and requires user approval per the explicit-permission rules. The plan includes a pause-for-approval task before running `gh repo create rrrozhd/zeroth-archive --public`.

### Archive Notice (ARCHIVE-03)

| Option | Description | Selected |
|--------|-------------|----------|
| Banner at top of archive repo README + GitHub description + `gh repo archive` (Recommended) | Multiple visibility points, uses locked phrasing | ✓ |
| README edit only | Less visible | |
| Repo description only | Not visible inside the repo | |

**Auto-selected:** Banner at top of archive repo README + GitHub description + `gh repo archive` (Recommended)
**Why recommended:** ARCHIVE-03 says "visible notice" — a single touchpoint fails that bar. The combination shows the notice in the README content, the repo list view, and the GitHub "archived" banner at the top of the repo page.

---

## PEP 420 Enforcement

| Option | Description | Selected |
|--------|-------------|----------|
| Verify via explicit `python -c` import tests + `grep` guardrails, delete top-level `__init__.py` (Recommended) | Mechanical, reproducible, no ambiguity | ✓ |
| Trust the rename script to have done it correctly | No evidence, no safety net | |

**Auto-selected:** Verify via explicit `python -c` import tests + `grep` guardrails (Recommended)
**Why recommended:** PEP 420 behavior is silent — if you leave a stray `__init__.py` at `src/zeroth/`, Python treats `zeroth` as a regular package, and the rename looks fine but silently prevents `zeroth.studio` / `zeroth.ext.*` from being added later. The explicit import tests and grep guardrails catch it.

---

## Claude's Discretion

The following were left as Claude's Discretion in CONTEXT.md because they depend on codebase discovery the researcher and planner will do:

- Exact CI wiring location for the interrogate gate (depends on existing CI layout)
- Whether to split docstring work into N subpackage commits or one commit per wave (depends on baseline delta)
- Whether to add `libcst` to `[dependency-groups].dev` or vendor the rename script as a standalone tool
- Exact artifact filenames inside `.planning/phases/27-*/artifacts/`
- Any additional `grep -v` exclusions for interrogate beyond migrations/tests/scripts/apps

---

## Deferred Ideas

- FUTURE-01 LibCST codemod as a shipped library module (Phase 27 ships only a one-shot script)
- Phase 28: PyPI publishing, optional-dep extras, CHANGELOG/LICENSE/CONTRIBUTING
- Phase 29: Studio Vue subtree extraction to `zeroth-studio` repo
- Phases 30–32: all docs work
- Later cleanup phase: eliminating pre-existing test skips
- FUTURE-04: docstring coverage badge in README
- Cascading `__init__.py` re-export refactor (explicitly rejected in PROJECT.md)

---

## Scope Creep Redirects

None surfaced during auto-selection. All decisions stayed within the ARCHIVE-01/02/03 + RENAME-01/02/03/04/05 boundary defined by the roadmap.

---

*Generated via `/gsd:discuss-phase 27 --auto`. See 27-CONTEXT.md for the decisions that downstream agents will act on.*
