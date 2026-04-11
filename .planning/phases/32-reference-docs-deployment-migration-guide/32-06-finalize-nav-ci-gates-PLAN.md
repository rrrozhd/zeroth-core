---
phase: 32-reference-docs-deployment-migration-guide
plan: 06
type: execute
wave: 3
depends_on:
  - "32-01-python-api-reference-mkdocstrings-PLAN"
  - "32-02-http-api-reference-swagger-PLAN"
  - "32-03-configuration-reference-dump-config-PLAN"
  - "32-04-deployment-guide-PLAN"
  - "32-05-migration-guide-PLAN"
files_modified:
  - mkdocs.yml
  - .github/workflows/docs.yml
autonomous: true
requirements:
  - DOCS-07
  - DOCS-08
  - DOCS-09
  - DOCS-10
  - DOCS-11
must_haves:
  truths:
    - "mkdocs.yml nav includes the 6 deployment pages under How-to → Deployment and the migration page under How-to → Migration"
    - "`uv run mkdocs build --strict` passes end-to-end with every Phase 32 page reachable from nav"
    - "CI (.github/workflows/docs.yml) runs `scripts/dump_openapi.py --check` and fails on OpenAPI drift"
    - "CI runs `scripts/dump_config.py --check` and fails on configuration drift"
    - "CI also copies openapi/zeroth-core-openapi.json → docs/assets/openapi/zeroth-core-openapi.json and diffs them to prevent asset drift"
    - "The existing strict mkdocs build step is preserved and runs after drift checks so failures surface early"
  artifacts:
    - path: "mkdocs.yml"
      provides: "Final nav with Phase 32 entries wired in"
      contains: "deployment/local-dev.md"
    - path: ".github/workflows/docs.yml"
      provides: "Drift-gated docs build"
      contains: "dump_openapi.py --check"
  key_links:
    - from: ".github/workflows/docs.yml"
      to: "scripts/dump_openapi.py"
      via: "uv run python step"
      pattern: "dump_openapi.py --check"
    - from: ".github/workflows/docs.yml"
      to: "scripts/dump_config.py"
      via: "uv run python step"
      pattern: "dump_config.py --check"
---

<objective>
Finalize Phase 32 by wiring the new Deployment + Migration pages into `mkdocs.yml` nav, extending `.github/workflows/docs.yml` with drift gates for OpenAPI and Configuration, keeping the strict mkdocs build gate, and running a final end-to-end verification of the docs site.

Purpose: Closes the phase. Ensures every page from Plans 32-01..05 is reachable, nothing has drifted, and the docs build is green on main and on PRs.

Output: Updated `mkdocs.yml` nav (Deployment section under How-to, Migration page under How-to), updated `.github/workflows/docs.yml` with three new steps (openapi drift, config drift, asset copy sync).
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-CONTEXT.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-01-python-api-reference-mkdocstrings-PLAN.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-02-http-api-reference-swagger-PLAN.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-03-configuration-reference-dump-config-PLAN.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-04-deployment-guide-PLAN.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-05-migration-guide-PLAN.md

@mkdocs.yml
@.github/workflows/docs.yml
</context>

<tasks>

<task type="auto">
  <name>Task 1: Wire Deployment + Migration pages into mkdocs.yml nav</name>
  <files>mkdocs.yml</files>
  <action>
    Per D-08:

    Under `nav:` → `How-to Guides:`, after the `Cookbook:` block, add two new entries:

    ```yaml
        - Deployment:
          - how-to/deployment/index.md
          - Local development: how-to/deployment/local-dev.md
          - Docker Compose: how-to/deployment/docker-compose.md
          - Standalone service: how-to/deployment/standalone-service.md
          - Embedded library: how-to/deployment/embedded-library.md
          - With Regulus: how-to/deployment/with-regulus.md
        - Migration from monolith: how-to/migration-from-monolith.md
    ```

    Indentation must match the existing `Subsystems:` and `Cookbook:` siblings in the `How-to Guides:` block. Do NOT touch the `Reference:` block — Plan 32-01 already wired Python API pages there.

    Run `uv run mkdocs build --strict` and fix any remaining warnings (broken internal links from cross-links in the deployment/migration pages).

    Expected warnings that should be resolved:
    - Deployment pages link to `../../reference/python-api/<x>.md` — these exist after Plan 32-01.
    - Migration page links to `deployment/docker-compose.md` — relative to `how-to/`.
    - Deployment with-regulus page links to `../cookbook/budget-cap.md` — already exists from Phase 31.
  </action>
  <verify>
    <automated>uv run mkdocs build --strict 2>&1 | tee /tmp/gsd-32-06.log && ! grep -Ei "^(WARNING|ERROR)" /tmp/gsd-32-06.log</automated>
  </verify>
  <done>mkdocs.yml nav contains Deployment section and Migration entry under How-to Guides. `uv run mkdocs build --strict` passes with zero warnings/errors.</done>
</task>

<task type="auto">
  <name>Task 2: Extend .github/workflows/docs.yml with drift gates</name>
  <files>.github/workflows/docs.yml</files>
  <action>
    Per D-07:

    Modify `.github/workflows/docs.yml`. After the "Install docs dependencies" step and BEFORE "Build site (strict)", insert three new steps:

    ```yaml
          - name: Check OpenAPI snapshot is up to date
            run: |
              uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json

          - name: Check Configuration reference is up to date
            run: |
              uv run python scripts/dump_config.py --check --out docs/reference/configuration.md

          - name: Check docs OpenAPI asset mirrors root snapshot
            run: |
              diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json
    ```

    Note:
    - The drift gates run BEFORE the strict build so a drift failure surfaces as a targeted error rather than a generic build failure.
    - The third step (`diff -u`) enforces that the mirrored copy under `docs/assets/openapi/` (introduced in Plan 32-02) stays in sync with the root snapshot. If it drifts, the user must re-run `cp openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json`.
    - The asset-copy step could instead be "auto-copy then check clean" — we choose strict diff to force the update to live in the PR that regenerated the snapshot, rather than silently fixing up CI state.
    - `uv sync --extra docs` (existing step) already installs pydantic/pydantic-settings (they're in core deps), so the config drift check needs no additional installs.
    - The `Install docs dependencies` step should be changed to `uv sync --extra docs` → `uv sync --extra docs` (unchanged) — the core deps install alongside because `uv sync` resolves the whole project. If drift-check imports fail due to missing runtime deps, change the step to `uv sync --all-extras` as a fallback; prefer the minimal install first.

    The existing "Build site (strict)" and "Deploy to GitHub Pages" steps stay exactly as-is (unchanged, just pushed down).
  </action>
  <verify>
    <automated>grep -q "dump_openapi.py --check" .github/workflows/docs.yml && grep -q "dump_config.py --check" .github/workflows/docs.yml && grep -q "diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json" .github/workflows/docs.yml && grep -q "mkdocs build --strict" .github/workflows/docs.yml</automated>
  </verify>
  <done>.github/workflows/docs.yml has three new drift-check steps before the strict build, strict build and deploy steps are preserved, all four references (openapi check, config check, asset diff, strict build) are present.</done>
</task>

<task type="auto">
  <name>Task 3: Final phase verification (local equivalent of CI)</name>
  <files></files>
  <action>
    Run the full CI sequence locally to prove the phase is shippable:

    ```bash
    uv sync --extra docs
    uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json
    uv run python scripts/dump_config.py --check --out docs/reference/configuration.md
    diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json
    uv run mkdocs build --strict
    ```

    All five commands must exit 0. Then spot-check the built site:

    ```bash
    # 20 python-api pages exist in the built site
    ls site/reference/python-api/*/index.html | wc -l
    # Expect: >= 20

    # Swagger UI embedded
    grep -q "swagger-ui-bundle.js" site/reference/http-api/index.html && echo OK

    # Configuration reference has at least 13 section headings
    grep -c "^<h2" site/reference/configuration/index.html || grep -c 'class="headerlink"' site/reference/configuration/index.html

    # Deployment pages
    ls site/how-to/deployment/*/index.html | wc -l
    # Expect: >= 6

    # Migration page
    test -f site/how-to/migration-from-monolith/index.html && echo OK
    ```

    If any step fails, fix the offending plan's output and re-run. Do NOT declare the phase complete until the five CI commands are all green.
  </action>
  <verify>
    <automated>uv sync --extra docs && uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json && uv run python scripts/dump_config.py --check --out docs/reference/configuration.md && diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json && uv run mkdocs build --strict && test -f site/reference/http-api/index.html && test -f site/reference/configuration/index.html && test -f site/how-to/migration-from-monolith/index.html && test -f site/how-to/deployment/local-dev/index.html</automated>
  </verify>
  <done>All drift checks pass, strict mkdocs build passes, every Phase 32 page renders to HTML, no broken links, phase ready for verification.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build --strict` exits 0
- `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` exits 0
- `uv run python scripts/dump_config.py --check --out docs/reference/configuration.md` exits 0
- `diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json` exits 0
- `.github/workflows/docs.yml` contains all three drift-check steps before the strict build
- `site/reference/python-api/graph/index.html`, `site/reference/http-api/index.html`, `site/reference/configuration/index.html`, `site/how-to/deployment/local-dev/index.html`, `site/how-to/migration-from-monolith/index.html` all exist
</verification>

<success_criteria>
Phase 32 complete: DOCS-07, DOCS-08, DOCS-09, DOCS-10, DOCS-11 all satisfied, CI gates drift on every PR, `mkdocs build --strict` is green, and the Reference + How-to (Deployment + Migration) quadrants of the docs site are fully populated.
</success_criteria>

<output>
After completion, create `.planning/phases/32-reference-docs-deployment-migration-guide/32-06-SUMMARY.md`
</output>
