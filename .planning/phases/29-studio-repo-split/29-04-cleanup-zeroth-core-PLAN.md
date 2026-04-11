---
phase: 29-studio-repo-split
plan: 04
type: execute
wave: 4
depends_on: [29-03]
files_modified:
  - apps/studio/   # deleted (entire tree)
  - apps/studio-mockups/   # deleted (entire tree)
  - tests/studio/   # deleted (empty dir with stale .pyc)
  - apps/   # deleted if empty after the above
  - README.md
  - pyproject.toml   # only if testpaths or tool configs reference studio
autonomous: true
requirements:
  - STUDIO-03
  - STUDIO-04

must_haves:
  truths:
    - "apps/studio/, apps/studio-mockups/, and tests/studio/ no longer exist in zeroth-core"
    - "tests/test_studio_api.py is untouched and still passes (per 29-RESEARCH critical finding)"
    - "README.md in zeroth-core has a 'Studio' section linking to https://github.com/rrrozhd/zeroth-studio"
    - "uv run pytest passes with no new failures vs. baseline"
    - "uv run ruff check src/ passes (deletion is source-only; no ruff impact expected)"
    - "pyproject.toml / pytest config do not reference the deleted paths"
  artifacts:
    - path: "README.md"
      provides: "zeroth-core README with Studio cross-link section"
      contains: "zeroth-studio"
  key_links:
    - from: "README.md"
      to: "https://github.com/rrrozhd/zeroth-studio"
      via: "markdown link in new Studio section"
      pattern: "github\\.com/rrrozhd/zeroth-studio"
---

<objective>
Wave 4: Delete the studio source from zeroth-core and add the cross-link. Per D-03, the split must land atomically — no transitional duplication. This plan runs AFTER Plan 03 has pushed zeroth-studio and CI is green, so zeroth-core losing the files is safe.

**CRITICAL: Preserve tests/test_studio_api.py.** Per 29-RESEARCH §Cross-Reference Audit, this file is a Python server-side contract test for the studio_api router. It imports zeroth.core.* and MUST NOT be deleted. Only `tests/studio/` (the empty bytecode-only directory) is removed.

Output: A clean zeroth-core that no longer contains the frontend, with a README pointer to the new repo and a still-green test suite.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/29-studio-repo-split/29-CONTEXT.md
@.planning/phases/29-studio-repo-split/29-RESEARCH.md
@.planning/phases/29-studio-repo-split/29-03-SUMMARY.md
@CLAUDE.md
@README.md
@pyproject.toml

<interfaces>
<!-- Paths to delete (VERIFIED from 29-RESEARCH): -->
- apps/studio/                 # real Vue app, now lives in zeroth-studio
- apps/studio-mockups/         # mockups, now live in zeroth-studio
- tests/studio/                # empty bytecode-only directory
- apps/                        # delete if empty after the above

<!-- Paths to NOT delete: -->
- tests/test_studio_api.py     # server-side Python contract test, imports zeroth.core.*
- scripts/dump_openapi.py      # added in Plan 01, reusable by Phase 32
- openapi/zeroth-core-openapi.json  # committed snapshot, reusable by Phase 32
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Pre-check zeroth-studio exists and is CI-green, then delete apps/ and tests/studio/ from zeroth-core</name>
  <files>apps/studio/, apps/studio-mockups/, tests/studio/, apps/, pyproject.toml</files>
  <action>
**Working directory: /Users/dondoe/coding/zeroth**

1. **Safety gate — do not delete unless zeroth-studio is on GitHub and green:**
```bash
# Verify the remote repo exists, is public, and has a green main
gh repo view rrrozhd/zeroth-studio --json name,isPrivate,defaultBranchRef \
  | python -c "import json,sys; d=json.load(sys.stdin); assert d['name']=='zeroth-studio' and d['isPrivate'] is False and d['defaultBranchRef']['name']=='main'; print('OK remote')"

CONCL=$(gh run list --repo rrrozhd/zeroth-studio --branch main --limit 1 --json conclusion --jq '.[0].conclusion')
[ "$CONCL" = "success" ] || { echo "REFUSING TO DELETE: zeroth-studio CI is not green (conclusion=$CONCL)"; exit 1; }
echo "OK: zeroth-studio CI green, safe to delete source from zeroth-core"
```

If this gate fails, STOP. The remote repo is the new source of truth — we don't delete locally until it's proven healthy.

2. **Confirm tests/test_studio_api.py exists at tests/ root (not inside tests/studio/):**
```bash
test -f tests/test_studio_api.py && echo "OK: test_studio_api.py present at tests/ root"
ls tests/studio/ 2>/dev/null || echo "tests/studio already absent"
# If tests/studio exists, inspect contents — should be only __pycache__
find tests/studio -type f -not -path '*/__pycache__/*' 2>/dev/null
# If this returns any .py sources (unexpected), STOP and re-read the directory
```

3. **Delete the three paths:**
```bash
git rm -rf apps/studio
git rm -rf apps/studio-mockups
# tests/studio may be untracked (only __pycache__ remains); git rm may fail — use rm -rf as fallback
git rm -rf tests/studio 2>/dev/null || rm -rf tests/studio

# If apps/ is now empty, remove it
if [ -d apps ] && [ -z "$(ls -A apps 2>/dev/null)" ]; then
  rmdir apps
fi

# Verify
ls apps 2>/dev/null && echo "apps/ still exists (non-empty?)" || echo "OK: apps/ removed"
ls tests/studio 2>/dev/null && echo "tests/studio still exists" || echo "OK: tests/studio removed"
test -f tests/test_studio_api.py && echo "OK: test_studio_api.py preserved"
```

4. **Scan pyproject.toml and other config for stale references:**
```bash
grep -n "apps/studio\|tests/studio" pyproject.toml pytest.ini setup.cfg 2>/dev/null || echo "No config refs"
```
If any match returns (other than comments that explicitly document the split), remove them. Likely candidates:
- `[tool.pytest.ini_options] testpaths` — should already be `["tests"]`, not `["tests", "tests/studio"]`
- `[tool.ruff]` includes/excludes
- `[tool.coverage.run] source` or `omit` entries

Do NOT blindly `sed` — read the file, understand the entry, make a targeted change.

5. **Broader repo scan for stray references** (per 29-RESEARCH Assumption A6):
```bash
grep -rn --include='*.md' --include='*.toml' --include='*.yml' --include='*.yaml' --include='*.sh' --include='*.py' \
  -e 'apps/studio' -e 'tests/studio' \
  . 2>/dev/null | grep -v '^./.planning/' | grep -v '^./.git/'
```
Expected matches: NONE outside `.planning/`. Anything found must be reviewed:
- Docs references → update to point at zeroth-studio or delete
- Script references → update or delete
- CI workflow references (`.github/workflows/`) → update or remove the step

6. **Run the full Python test suite to prove nothing broke:**
```bash
uv run pytest -v --no-header -ra
uv run ruff check src/
```
Both must pass. `tests/test_studio_api.py` specifically must still pass — it's the canonical way to verify D-03 side effects.
  </action>
  <verify>
    <automated>test ! -d apps/studio && test ! -d apps/studio-mockups && test ! -d tests/studio && test -f tests/test_studio_api.py && uv run pytest -v tests/test_studio_api.py && uv run ruff check src/</automated>
  </verify>
  <done>apps/studio, apps/studio-mockups, tests/studio are gone; tests/test_studio_api.py is intact; zeroth-studio remote CI is green; pytest + ruff pass; no stray references to the deleted paths outside .planning/.</done>
</task>

<task type="auto">
  <name>Task 2: Add "Studio" cross-link section to zeroth-core README.md</name>
  <files>README.md</files>
  <action>
**Working directory: /Users/dondoe/coding/zeroth**

Per D-08, add a short "Studio" section to the zeroth-core README.md pointing at the new frontend repo.

1. Read the current README.md to find a sensible insertion point (typically after the project blurb, before the Getting Started or Install section — or as a top-level section near the end if the README is thin).

2. Insert the following section (adapt surrounding markdown to match the README's style):

```markdown
## Studio

Zeroth's canvas UI for authoring and inspecting workflows lives in a separate repo:

**[rrrozhd/zeroth-studio](https://github.com/rrrozhd/zeroth-studio)** — Vue 3 + Vue Flow frontend that speaks to `zeroth-core` over HTTP.

The studio was split out in v3.0 Phase 29 to let the two projects ship on independent release cadences. A cross-repo [compatibility matrix](https://github.com/rrrozhd/zeroth-studio#compatibility) documents which studio versions pair with which core versions.
```

3. If the README already has a "Related projects" or "Ecosystem" section, fold this into that section instead of adding a duplicate top-level heading.

4. If the README is minimal/placeholder (the file is 1-3 lines or missing), still add a proper section — prefer a small self-contained block over a one-liner.

5. Verify the link is correct:
```bash
grep -n "github.com/rrrozhd/zeroth-studio" README.md
```
Should return at least one match.
  </action>
  <verify>
    <automated>grep -q "github.com/rrrozhd/zeroth-studio" README.md && grep -qi "studio" README.md</automated>
  </verify>
  <done>README.md has a Studio section linking to https://github.com/rrrozhd/zeroth-studio with a one-line description of what the studio provides.</done>
</task>

</tasks>

<verification>
- apps/studio/, apps/studio-mockups/, tests/studio/ deleted from zeroth-core
- tests/test_studio_api.py present and passing
- uv run pytest full suite passes with no new failures
- uv run ruff check src/ passes
- README.md cross-links to https://github.com/rrrozhd/zeroth-studio
- No stray references to deleted paths outside .planning/
- pyproject.toml / pytest config updated if it previously referenced studio paths

Phase-level verification (all plans combined):
- STUDIO-01: rrrozhd/zeroth-studio exists with preserved history (Plan 02)
- STUDIO-02: zeroth-studio CI green on main (Plan 03)
- STUDIO-03: zero Python imports under apps/ in zeroth-studio (Plan 03), zero apps/ remaining in zeroth-core (this plan)
- STUDIO-04: compat matrix in zeroth-studio README (Plan 03), cross-link in zeroth-core README (this plan)
- STUDIO-05: types.gen.ts + drift gate on zeroth-studio (Plan 03)
</verification>

<success_criteria>
1. Single-source-of-truth achieved: studio lives only in zeroth-studio, zeroth-core has no frontend source
2. zeroth-core test suite stays green (pytest + ruff)
3. tests/test_studio_api.py preserved as the server-side contract for the studio_api router
4. README cross-link closes the STUDIO-04 loop (matrix on the dependent side, pointer from the dependency side)
5. Safety gate (remote+CI check) prevents accidental deletion if Plan 03 didn't finish cleanly
</success_criteria>

<output>
After completion, create `.planning/phases/29-studio-repo-split/29-04-SUMMARY.md` covering:
- Files/directories deleted (exact paths + counts)
- pyproject.toml diffs (if any) — what was referenced, how it was updated
- Test suite results before and after deletion (counts of passed/failed/skipped)
- README.md insertion point and before/after snippet
- Any stray references found in the broader scan and how they were resolved
- Confirmation that tests/test_studio_api.py is untouched

Also write `.planning/phases/29-studio-repo-split/29-SUMMARY.md` as the phase-level rollup covering all four plans, with links to each plan's SUMMARY and a final STUDIO-01..STUDIO-05 status table.
</output>
