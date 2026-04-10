# Phase 27: Monolith Archive & Namespace Rename - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning
**Source:** discuss-phase `--auto` (recommended defaults auto-selected; see 27-DISCUSSION-LOG.md)

<domain>
## Phase Boundary

Phase 27 delivers two things that must land in order:

1. **Monolith preservation.** Before touching a single file, capture the current monolithic Zeroth repo in three independent archive layers (local tarball, local bare mirror, GitHub `rrrozhd/zeroth-archive`), with every branch, stash, and the detached-HEAD worktree recoverable from the mirror.
2. **Pure namespace rename.** Relocate all Python source from `zeroth.*` to `zeroth.core.*` under a PEP 420 namespace package (no top-level `zeroth/__init__.py`), rewrite every import / entry point / string reference, update packaging metadata, land ≥90% Google-style docstring coverage on the new public surface, and prove the full existing test suite still passes with zero regressions.

This phase is **pure packaging/structural work** — zero new runtime features, zero deletions of functionality, zero behavioral changes. If a line of code does something different after the rename than before, the rename is wrong.

This phase does NOT deliver:
- PyPI publishing (Phase 28)
- `econ-instrumentation-sdk` PyPI migration (Phase 28 — for now the `file://` dep stays)
- `zeroth-studio` repo split (Phase 29)
- Docs site infrastructure or content (Phases 30–32)

The `/tmp/zeroth-split/zeroth-core-build/` scratch tree already contains a hand-executed version of the rename. This phase does **not** adopt that tree wholesale — it is reference-only. The actual rename is performed in-place against this repo to preserve git history.

</domain>

<decisions>
## Implementation Decisions

### Rename Execution Strategy (RENAME-01, RENAME-02, RENAME-03)

- **D-01:** The rename is performed **in-place against this repo** using `git mv`, NOT by copying the `/tmp/zeroth-split/zeroth-core-build/` tree over. Rationale: preserves git history + blame for every moved file. The `/tmp` tree is consulted as a reference to cross-check the final layout after the rename.
- **D-02:** Rename procedure:
  1. `git mv src/zeroth/<each subpkg>/ src/zeroth/core/<each subpkg>/` for every current subpackage (`agent_runtime`, `approvals`, `audit`, `conditions`, `config`, `contracts`, `demos`, `deployments`, `dispatch`, `econ`, `execution_units`, `graph`, `guardrails`, `identity`, `mappings`, `memory`, `migrations`, `observability`, `orchestrator`, `policy`, `runs`, `sandbox_sidecar`, `secrets`, `service`, `storage`, `studio`, `webhooks`).
  2. `git rm src/zeroth/__init__.py` (final step that flips on PEP 420 behavior).
  3. Create `src/zeroth/core/__init__.py` as empty (or with version export) — a regular package, since we want `import zeroth.core` to resolve. `zeroth` itself becomes the PEP 420 namespace package.
- **D-03:** Import rewrite uses **`libcst`** (AST-level, preserves formatting) for all `.py` files, NOT regex sed. Covered transformations:
  - `import zeroth` → `import zeroth.core`
  - `import zeroth.X` → `import zeroth.core.X`
  - `from zeroth import X` → `from zeroth.core import X`
  - `from zeroth.X import Y` → `from zeroth.core.X import Y`
  - `importlib.import_module("zeroth.X")` → `importlib.import_module("zeroth.core.X")`
- **D-04:** Non-Python text (`pyproject.toml`, `alembic.ini`, YAML configs, `Dockerfile`, shell scripts, markdown docs inside the repo, entry-point strings, `__main__` console scripts) is rewritten by a scripted `sed`-based pass with an explicit include/exclude list. Rename script is committed under `scripts/rename_to_zeroth_core.py` so the transformation is reproducible and reviewable.
- **D-05:** The rename script lives in the repo under `scripts/` (not `zeroth.core.codemods` — that is `FUTURE-01`, explicitly deferred). It is a one-shot migration script, not a shipped library.
- **D-06:** Studio Python subtree (`src/zeroth/studio/`) moves under `src/zeroth/core/studio/` along with everything else (pure-rename mandate — zero deletions). Phase 29 will extract it via `git filter-repo`; Phase 27 does not touch it structurally beyond the rename.

### Commit Granularity

- **D-07:** The rename lands as **multiple atomic commits**, one per logical step, so each is reviewable in isolation:
  1. `chore(27): convert detached HEAD + stashes to archive branches for mirror preservation`
  2. `chore(27): archive monolith (tarball + bare mirror + GitHub push)` (or three separate commits, one per layer)
  3. `refactor(27): git mv src/zeroth/<subpkgs>/ → src/zeroth/core/<subpkgs>/` (file moves only, no content changes)
  4. `refactor(27): rewrite imports from zeroth.* to zeroth.core.*` (libcst pass)
  5. `refactor(27): rewrite non-Python references (pyproject, alembic, YAML, Dockerfile, entry points)`
  6. `refactor(27): drop top-level src/zeroth/__init__.py to enable PEP 420 namespace`
  7. `chore(27): update pyproject.toml package name to zeroth-core and wheel target`
  8. `test(27): verify full suite passes on renamed layout`
  9. `docs(27): add interrogate config (Google style, fail-under=90) + module docstrings (wave N)`  ← likely split across multiple commits
- **D-08:** Commits are sequenced so that at every commit boundary, **the repo is importable** (no half-renamed state on `main`). The file-move commit (#3) and the import-rewrite commit (#4) MUST be squashed or amended into a single commit if CI runs per-commit — otherwise commit #3 leaves the tree broken.
  - **Locked:** Execute as a single commit for steps 3+4+6 combined: `refactor(27): relocate zeroth.* to zeroth.core.* (git mv + import rewrite + drop top-level __init__)`. Non-Python reference rewrite (#5) and pyproject update (#7) may be a separate commit since they do not break imports.

### Packaging Metadata (touches PKG territory but required for RENAME to load)

- **D-09:** `pyproject.toml` `[project].name` changes from `"zeroth"` to `"zeroth-core"` in Phase 27. Publishing to PyPI remains Phase 28 — Phase 27 only renames the declared package name so editable installs and test discovery work.
- **D-10:** `[tool.hatch.build.targets.wheel]` (or equivalent build-target config) is set to `packages = ["src/zeroth/core"]`. The `src/zeroth/` directory has no `__init__.py` and is NOT a wheel package — hatchling treats `zeroth.core` as the installable package and `zeroth` as an implicit namespace package resolved at import time.
- **D-11:** Version stays at `0.1.0` in Phase 27. Versioning/publishing bumps happen in Phase 28.
- **D-12:** The `econ-instrumentation-sdk @ file:///Users/dondoe/coding/regulus/sdk/python` dependency is **left untouched** in Phase 27 — the PyPI swap is PKG-01 (Phase 28). Phase 27 does not modify `dependencies`, `[project.optional-dependencies]`, or `[project.scripts]` beyond fixing any entry-point module paths that reference `zeroth.<X>` (they become `zeroth.core.<X>`).
- **D-13:** Any `[project.scripts]` console scripts that point at `zeroth.<module>:<fn>` are rewritten to `zeroth.core.<module>:<fn>` as part of commit #5.

### Docstring Coverage (RENAME-05)

- **D-14:** Tool: `interrogate` (already hinted in the requirement). Configuration lives in `pyproject.toml` under `[tool.interrogate]` with at minimum: `fail-under = 90`, `style = "google"`, `exclude = ["src/zeroth/core/migrations", "tests", "scripts", "apps"]`, `ignore-init-method = true`, `ignore-init-module = true`, `ignore-nested-functions = true`.
- **D-15:** Docstring **style enforcement** uses `ruff` with `[tool.ruff.lint.pydocstyle] convention = "google"` and the `D` rule family enabled for `src/zeroth/core/**`. Tests, migrations, scratch scripts, and apps are exempted.
- **D-16:** Baseline-first workflow: before writing any new docstrings, run `uv run interrogate -v src/zeroth/core/ --fail-under 0` to get the current coverage number. Record the baseline in the PROGRESS log so we know how much ground we are covering.
- **D-17:** Docstrings are **human-reviewed, not LLM auto-generated.** For the public API surface, the planner allocates explicit tasks per subpackage so an implementer (human or agent) writes Google-style docstrings that reflect actual behavior. Placeholder docstrings (`"""TODO."""`, `"""Do the thing."""`) are forbidden — interrogate counts them but they fail the quality intent.
- **D-18:** Prioritization order for filling docstring gaps (if baseline is below 90%): (1) top-level package `__init__.py` modules and their public `__all__`, (2) class/function signatures in the public HTTP service layer (`service/`, `deployments/`), (3) `orchestrator/`, `graph/`, `execution_units/`, `agent_runtime/`, (4) everything else. If baseline is already ≥90%, the task becomes "verify, do not regress, land the interrogate CI gate."
- **D-19:** CI enforcement: after Phase 27 lands, `uv run interrogate src/zeroth/core` runs in CI and is a hard gate. Add to whatever check command the repo uses (likely a `make check` or a `scripts/check.sh` — planner to discover). **Claude's Discretion** on exact CI wiring since it depends on existing CI layout.

### Test Suite Verification (RENAME-04)

- **D-20:** Baseline command: `uv run pytest -v --no-header -ra 2>&1 | tee .planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/pytest-before-rename.log` run **before the rename touches any file**. Captures test count, any existing skips, any existing xfails.
- **D-21:** Post-rename verification: same command, output to `pytest-after-rename.log`. Diff against baseline. Required invariants: same number of collected tests, same passed count, no new failures, no new errors, no new skips.
- **D-22:** Requirement text says "zero skips." **Interpretation:** zero NEW skips introduced by the rename. Pre-existing skips (e.g., `pytest.skip("requires docker-compose")`, provider-dependent skips) are grandfathered IF they are already present in the baseline log. The planner must verify the baseline explicitly — if there are many pre-existing skips, flag as a research question before executing the rename. **Locked:** grandfathered skips allowed only if they existed pre-rename; no new skips allowed; any newly-skipped test is treated as a regression.
- **D-23:** Environment consistency: the baseline and post-rename runs use the same Python version (`.python-version` or `uv python pin`), the same `uv.lock`, and the same service dependencies (Postgres, Redis, Docker). Tests that depend on external services follow whatever the existing repo convention is for dev-mode skipping.

### Archive Mechanics (ARCHIVE-01, ARCHIVE-02, ARCHIVE-03)

- **D-24:** Archive **precedes** the rename. Order: (A) branch/stash preservation → (B) tarball → (C) bare mirror → (D) push bare mirror to `rrrozhd/zeroth-archive` → (E) verify recoverability → THEN rename work starts.
- **D-25:** **Branch/stash preservation** (prerequisite for the mirror to actually capture everything):
  - Enumerate stashes with `git stash list` (expected: 2). For each stash, create a named ref: `git branch archive/stash-0 stash@{0}`, `git branch archive/stash-1 stash@{1}`, so they become normal refs the mirror will capture.
  - Enumerate worktrees with `git worktree list` (expected: 36 + 1 detached). For the detached-HEAD worktree, create a named branch `archive/detached-wt-<shortsha>` at its HEAD so it is not GC-eligible.
  - Verify with `git for-each-ref refs/heads/archive/` that all the synthesized refs exist before cloning the mirror.
- **D-26:** **Tarball layer:**
  - Path: `$HOME/archives/zeroth-monolith/zeroth-monolith-2026-04-10.tar.gz`
  - Contents: full working tree of this repo (the uncommitted state does not matter — tarball is a safety net snapshot).
  - Command: `tar --exclude-vcs-ignores --exclude='.venv' --exclude='__pycache__' --exclude='.pytest_cache' --exclude='node_modules' -czf <path>`
  - Include `.git/` directory in the tarball so the snapshot is self-contained and not just a working-tree copy.
  - Record the resulting file size + sha256 in the PROGRESS log as evidence.
- **D-27:** **Bare mirror layer:**
  - Path: `$HOME/archives/zeroth-monolith/zeroth-monolith.git/`
  - Command: `git clone --mirror /Users/dondoe/coding/zeroth $HOME/archives/zeroth-monolith/zeroth-monolith.git`
  - Verify: `cd $HOME/archives/zeroth-monolith/zeroth-monolith.git && git for-each-ref | wc -l` — should list all 84 branches + the synthesized `archive/*` refs + tags + stashes-as-branches.
  - Verify recoverability of the detached-HEAD worktree: `git -C $HOME/archives/zeroth-monolith/zeroth-monolith.git log archive/detached-wt-<shortsha> -1`.
- **D-28:** **GitHub layer:**
  - Target repo: `rrrozhd/zeroth-archive` (user-owned).
  - Repo creation is a **user action** — the plan includes an explicit "USER ACTION REQUIRED" checkpoint asking the user to confirm they want Claude to invoke `gh repo create rrrozhd/zeroth-archive --public --description "Archived monolithic Zeroth repo — see rrrozhd/zeroth-core and rrrozhd/zeroth-studio"` OR to create the repo manually and grant push access.
  - Push: from the bare mirror dir, `git remote add origin git@github.com:rrrozhd/zeroth-archive.git && git push --mirror origin`.
  - `--mirror` push sends all refs including the synthesized `archive/*` branches. Verify on GitHub UI that branch count matches local mirror.
- **D-29:** **Archive notice (ARCHIVE-03):**
  - `README.md` in the archive repo (pushed via a single commit to `main`, since the mirrored repo has no `main` after the push): add a prominent banner at the top:
    > ⚠️ **This repository is archived.** Active development continues in `rrrozhd/zeroth-core` (Python library) and `rrrozhd/zeroth-studio` (Vue frontend). This repo exists only to preserve the pre-split monolith history.
  - GitHub repo description (set via `gh repo edit rrrozhd/zeroth-archive --description "..."`) repeats the same message.
  - Pin the archive notice at the top of the README. Leave all other archived content as-is (do not touch the old README body).
  - Archive the repo on GitHub (`gh repo archive rrrozhd/zeroth-archive`) so the UI shows the archived banner.
- **D-30:** **Recoverability acceptance test:** after all three layers are live, run a dry-run recovery to prove recoverability:
  - Clone the bare mirror to a scratch dir: `git clone $HOME/archives/zeroth-monolith/zeroth-monolith.git /tmp/zeroth-monolith-recovery-test`
  - Check out a known worktree branch AND the synthesized `archive/detached-wt-*` branch
  - Check out one of the `archive/stash-*` branches to verify stash content is intact
  - Run `git log --oneline -5` on each to prove history is present
  - Record the test output in the PROGRESS artifact. Drop the scratch clone after.

### PEP 420 Enforcement & Verification (RENAME-02)

- **D-31:** Verification commands that MUST pass post-rename (the planner turns these into acceptance criteria tasks):
  - `test ! -f src/zeroth/__init__.py` — no top-level init
  - `uv run python -c "import zeroth.core; print(zeroth.core.__path__)"` — subpackage resolves
  - `uv run python -c "import zeroth; print(zeroth.__path__); print(hasattr(zeroth, '__file__'))"` — namespace package has `__path__` but no `__file__`
  - `uv run python -c "from zeroth.core.orchestrator import GraphOrchestrator"` (or any well-known public class) — end-to-end import works
- **D-32:** Explicit guardrail: `grep -rn "from zeroth import" src/ tests/ scripts/ apps/ 2>/dev/null` MUST return zero matches after the rename. Same for `from zeroth\.\w` that does not start with `from zeroth.core.`.
- **D-33:** `src/zeroth/` directory may contain ONLY the `core/` subdirectory and NOTHING ELSE after the rename (no `__init__.py`, no stray modules, no `py.typed` at the `zeroth/` level — `py.typed` lives under `src/zeroth/core/py.typed` if present).

### What Phase 27 Does NOT Touch

- **D-34:** `econ-instrumentation-sdk` file-path dep (Phase 28).
- **D-35:** Any PyPI publishing config, GitHub Actions for release, trusted publisher (Phase 28).
- **D-36:** Studio frontend Vue code or its repo split (Phase 29).
- **D-37:** Docs site scaffolding, mkdocs config, Diátaxis IA (Phases 30–32).
- **D-38:** New runtime features — any behavior change is a bug in the rename.
- **D-39:** Adding new tests (only reshaping imports in existing tests). Docstrings are the only "new content" allowed in Phase 27.

### Claude's Discretion

- Exact CI wiring location for the interrogate gate (existing CI layout determines this).
- Whether to split docstring-coverage work into N subpackage-scoped commits or one commit per wave — depends on baseline delta.
- Whether `libcst` is already a dev dep; if not, add it to `[dependency-groups].dev` for the rename script (dev-only, no runtime impact).
- Exact output directory for pytest logs inside `.planning/phases/27-*/artifacts/` — planner picks filenames.
- Whether the rename script lives as a `.py` file or a small cli in `scripts/` — implementer picks.
- Any additional `grep -v` exclusions for interrogate (e.g., generated migration scripts) beyond the ones listed in D-14.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Scope Anchors
- `.planning/ROADMAP.md` §Phase 27 — goal + 5 success criteria are the hard scope boundary
- `.planning/REQUIREMENTS.md` §Namespace Rename (RENAME-01 through RENAME-05) — acceptance criteria
- `.planning/REQUIREMENTS.md` §Monolith Archive (ARCHIVE-01, ARCHIVE-02, ARCHIVE-03) — acceptance criteria
- `.planning/PROJECT.md` §Key Decisions — the "pure rename, take EVERYTHING into zeroth.core.*" and "PEP 420 namespace" rows are locked milestone decisions

### Reference Implementation (consult, do NOT copy wholesale)
- `/tmp/zeroth-split/zeroth-core-build/` — hand-executed prior pass at the rename. Useful for cross-checking the final tree layout. NOT a source tree to copy over — the rename runs in-place in `/Users/dondoe/coding/zeroth/` so git history is preserved.
- `/tmp/zeroth-split/zeroth-core-build/src/zeroth/core/` — shows the expected 27-subpackage layout under `zeroth.core.*` (matches current `src/zeroth/` 1:1 except for the relocation)
- `/tmp/zeroth-split/zeroth-core-build/pyproject.toml` — reference for the renamed `name = "zeroth-core"` + `packages = ["src/zeroth/core"]` shape

### Current Repo State (to be transformed)
- `pyproject.toml` — current `[project].name = "zeroth"`, current hatchling config, current `dependencies` block (esp. `econ-instrumentation-sdk @ file://...` which stays)
- `src/zeroth/` — current layout with 27 top-level subpackages + `__init__.py`
- `src/zeroth/studio/` — backend Python for studio API (renames along with everything else; Phase 29 extracts it later)
- `tests/` — full existing test suite (~280 tests) that must still pass after rename
- `alembic.ini` — may contain `zeroth.` module references that need rewriting
- `Dockerfile`, `docker-compose.yml` — may reference `zeroth` module paths in commands/env

### External Specifications
- PEP 420 — Implicit Namespace Packages — defines why no `zeroth/__init__.py` is needed
- PEP 621 — Project metadata in pyproject.toml — defines the `[project]` table
- Hatchling docs — `[tool.hatch.build.targets.wheel]` layout for namespace packages
- `interrogate` docs — `[tool.interrogate]` config options (fail-under, style, exclude)
- `ruff` pydocstyle rules (D family) — Google convention enforcement
- `libcst` — AST-preserving Python code rewrites
- Git `clone --mirror` docs — mirror semantics for ref preservation
- `git filter-repo` docs — not used in Phase 27 but referenced by Phase 29 (do not pre-empt)
- Google Python Style Guide §3.8 — docstring format reference

### Iteration Log Convention
- `PROGRESS.md` (root) — CLAUDE.md mandates every meaningful unit of work updates this via the `progress-logger` skill. Phase 27 executors MUST follow.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `/tmp/zeroth-split/zeroth-core-build/` — an already-renamed copy to diff against for verification
- `uv` + `hatchling` + `ruff` + `pytest` toolchain is already configured — no new tools needed except `interrogate` and possibly `libcst` (dev-only)
- Existing `[build-system] requires = ["hatchling"]` handles src-layout packages cleanly with a small config tweak

### Established Patterns
- src-layout: `src/zeroth/<subpkg>/` — the rename keeps src-layout, just adds one level of nesting
- uv-based deps: `uv sync`, `uv run pytest` — these commands stay identical post-rename
- ruff for lint + format — existing config migrates by pointing at `src/zeroth/core` instead of `src/zeroth`
- Modular monolith: 27 top-level subpackages — every one of them renames, nothing is excluded or flattened

### Integration Points
- Any `from zeroth.X import Y` in tests/, scripts/, apps/, demos/ — all rewritten
- Console scripts in `[project.scripts]` — module paths rewritten
- Alembic migrations that reference Python module paths for metadata lookup
- Dockerfile `CMD`/`ENTRYPOINT` lines that may reference `python -m zeroth.X`
- `docker-compose.yml` service command fields
- GovernAI git-pinned dep — not affected by rename (external package)
- Regulus file-path dep — not affected by rename (external package)

### Scale Reality Checks
- 36 pre-existing worktrees → archive mirror must capture them all
- 2 existing stashes → must be converted to named refs before mirror
- 1 detached-HEAD worktree → must be converted to named ref before mirror
- 84 branches per `git branch -a | wc -l` → mirror must preserve all
- ~22K LOC Python source + ~18K LOC tests → libcst rewrite processes ~40K LOC

</code_context>

<specifics>
## Specific Ideas

- Archive directory hierarchy: `$HOME/archives/zeroth-monolith/` holds both the tarball and the `zeroth-monolith.git` bare mirror together, so a single directory is the "monolith preservation" artifact on disk.
- GitHub archive repo name is **locked** at `rrrozhd/zeroth-archive` (per PROJECT.md + REQUIREMENTS.md).
- The archive notice phrase is **locked** at: "archived — see rrrozhd/zeroth-core and rrrozhd/zeroth-studio" (per ARCHIVE-03 text).
- Docstring convention is **locked** at Google style (per RENAME-05 text).
- Docstring coverage threshold is **locked** at ≥90% (per RENAME-05 text).
- Test suite size is **locked** at 280+ tests (per RENAME-04 text) — post-rename count must be ≥ baseline count.
- The `/tmp/zeroth-split/zeroth-core-build/` scratch tree is **reference-only**. It informed the plan but is not adopted into this repo. Git history preservation is the reason.

</specifics>

<deferred>
## Deferred Ideas

- **FUTURE-01 (backlog):** LibCST codemod shipped as `zeroth.core.codemods.rename_from_monolith` for external consumers. Phase 27 only ships an in-repo one-shot script; turning it into a library-grade codemod is explicitly deferred per REQUIREMENTS.md.
- **Phase 28:** `econ-instrumentation-sdk` PyPI publishing + `zeroth-core` PyPI publishing + trusted publisher OIDC wiring + optional-dep extras + CHANGELOG/LICENSE/CONTRIBUTING files.
- **Phase 29:** Studio Vue subtree extraction to `rrrozhd/zeroth-studio` via `git filter-repo`. Phase 27 moves Studio Python under `zeroth.core.studio` — it does not touch Vue code layout or create the new repo.
- **Phases 30–32:** All docs site, mkdocs-material, Diátaxis IA, Concept/Usage pages, Cookbook, examples/, API reference, Deployment Guide, Migration Guide.
- **Deferred behavior change:** removing pre-existing test skips that are not introduced by the rename. Phase 27 grandfathers them; a later cleanup phase can attack them.
- **Deferred:** docstring coverage badge in README (FUTURE-04).
- **Deferred:** any refactor of `__init__.py` re-exports (PROJECT.md explicitly decided against the cascading __init__ refactor — pure rename only).

</deferred>

---

*Phase: 27-ship-zeroth-as-pip-installable-library-zeroth-core*
*Context gathered: 2026-04-10 via /gsd:discuss-phase 27 --auto*
