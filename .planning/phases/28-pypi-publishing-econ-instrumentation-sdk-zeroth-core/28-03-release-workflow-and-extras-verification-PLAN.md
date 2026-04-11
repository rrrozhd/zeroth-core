---
phase: 28
plan: 03
type: execute
wave: 2
depends_on:
  - 28-01
  - 28-02
files_modified:
  - .github/workflows/release-zeroth-core.yml
  - .github/workflows/verify-extras.yml
  - README.md
autonomous: false
requirements:
  - PKG-02
  - PKG-03
  - PKG-05
  - PKG-06
tags:
  - github-actions
  - trusted-publisher
  - oidc
  - pypi
  - ci

must_haves:
  truths:
    - ".github/workflows/release-zeroth-core.yml exists and is triggered by release.published events"
    - "Release workflow stages are: build → smoke-install → test-wheel → publish-testpypi → smoke-from-testpypi → publish-pypi (in that order, with explicit needs: dependencies)"
    - "publish-testpypi and publish-pypi jobs declare permissions: { id-token: write, contents: read } at JOB level, not workflow level"
    - "publish-testpypi uses environment: testpypi; publish-pypi uses environment: pypi (separate envs per trusted-publisher pattern)"
    - "Publish jobs use pypa/gh-action-pypi-publish@release/v1 (Sigstore attestations default-on in v1.14+)"
    - "Build job asserts git tag matches pyproject.toml [project].version, fails loudly on mismatch (D-07)"
    - "test-wheel job runs pytest against the installed wheel, not src mode, to catch packaging bugs"
    - "smoke-from-testpypi retries pip install in a loop to handle TestPyPI indexing lag (pitfall #4)"
    - "smoke-from-testpypi runs examples/hello.py if ANTHROPIC_API_KEY secret is present, skips cleanly otherwise"
    - ".github/workflows/verify-extras.yml exists — runs on every push/PR, matrix job per extra (memory-pg, memory-chroma, memory-es, dispatch, sandbox, all) that creates a clean venv, installs 'zeroth-core[<extra>]' from the in-tree source, and imports the modules that depend on it (per D-04)"
    - "README.md has an Install section (or equivalent) mentioning at least one extras example like `pip install 'zeroth-core[memory-pg]'`"
    - "User action checkpoint pauses execution so the user can register trusted publishers on pypi.org + test.pypi.org before the release workflow is first run"
  artifacts:
    - path: ".github/workflows/release-zeroth-core.yml"
      provides: "Trusted-publisher release pipeline for zeroth-core with TestPyPI staging and Sigstore attestations"
      contains: "pypa/gh-action-pypi-publish"
    - path: ".github/workflows/verify-extras.yml"
      provides: "Per-extra clean-venv install + import smoke CI matrix (PKG-03 gate per D-04)"
      contains: "matrix"
    - path: "README.md"
      provides: "Updated Install section referencing at least one extras example"
  key_links:
    - from: ".github/workflows/release-zeroth-core.yml publish-testpypi"
      to: "test.pypi.org trusted publisher"
      via: "OIDC token (id-token: write) + environment=testpypi"
      pattern: "environment:\\s*testpypi"
    - from: ".github/workflows/release-zeroth-core.yml publish-pypi"
      to: "pypi.org trusted publisher"
      via: "OIDC token (id-token: write) + environment=pypi"
      pattern: "environment:\\s*pypi"
    - from: "verify-extras.yml matrix"
      to: "each declared extra in pyproject.toml"
      via: "clean-venv pip install + module import"
      pattern: "zeroth-core\\["
    - from: "smoke-from-testpypi job"
      to: "examples/hello.py"
      via: "subprocess execution with ANTHROPIC_API_KEY gate"
      pattern: "examples/hello\\.py"
---

<objective>
Ship the two GitHub Actions workflows that make the Phase 28 release pipeline real: `release-zeroth-core.yml` (trusted-publisher OIDC release to PyPI with TestPyPI staging + Sigstore attestations, triggered by `release.published`) and `verify-extras.yml` (continuous CI matrix that clean-installs every declared extra). Then update README.md with a minimal Install section referencing at least one extra so users discover the feature, and pause for the one manual-only step of Phase 28: the user registering trusted publishers on pypi.org and test.pypi.org.

Purpose: PKG-05 requires trusted-publisher releases (no long-lived tokens). PKG-03 requires every extra provably installable — the verify-extras matrix is the continuous gate. PKG-06 is closed when the release workflow's smoke-from-testpypi step runs `examples/hello.py` end-to-end. The manual PyPI trusted-publisher configuration is the single remaining step that Claude literally cannot perform (the pypi.org web UI has no CLI), and it MUST be done before the release workflow is first invoked — hence the checkpoint.

Output: Two workflow files, a README.md diff, and a resumed session after the user registers publishers. The first actual release (creating a GitHub Release for `v0.1.1`) happens AFTER this plan completes and is documented as a post-phase follow-up in the phase SUMMARY — not part of this plan's success criteria.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-CONTEXT.md
@.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-RESEARCH.md
@.github/workflows/ci.yml
@README.md
@pyproject.toml
@CLAUDE.md

<prior_plan_outputs>
Plan 01 (28-01) produced:
  - pyproject.toml at version 0.1.1, Apache-2.0, six extras declared, py.typed included
  - dist/zeroth_core-0.1.1-py3-none-any.whl verified clean-venv installable
  - [tool.hatch.build.targets.wheel].packages still ["src/zeroth"] (pitfall #2 guard)

Plan 02 (28-02) produced:
  - LICENSE, CHANGELOG.md, CONTRIBUTING.md at repo root
  - examples/hello.py with env-gated ANTHROPIC_API_KEY and clean exit-0 skip path
  - Reconciled .planning/STATE.md blockers (Regulus entry removed)

This plan depends on both: the release workflow references `examples/hello.py` (from 02), builds `dist/*.whl` at version 0.1.1 (from 01), and the verify-extras matrix iterates the extras declared in pyproject.toml (from 01).
</prior_plan_outputs>

<interfaces>
<!-- Canonical release workflow pattern from RESEARCH.md lines 205-307. -->
<!-- Executor: this is not a literal paste — adapt with the corrections below. -->

Required workflow shape (release-zeroth-core.yml):
  trigger: on.release.types = [published]
  jobs in this order with explicit needs:
    1. build                 — uv build, tag-matches-version guard, upload dist/ artifact
    2. smoke-install         — download dist, clean venv, pip install wheel, import zeroth.core
    3. test-wheel            — download dist, clean venv, pip install wheel + pytest deps, run pytest against installed wheel
    4. publish-testpypi      — download dist, environment=testpypi, permissions.id-token=write (JOB LEVEL), pypa/gh-action-pypi-publish@release/v1 with repository-url=https://test.pypi.org/legacy/
    5. smoke-from-testpypi   — retry loop for indexing lag, pip install from test.pypi.org, env-gated run of examples/hello.py
    6. publish-pypi          — download dist, environment=pypi, permissions.id-token=write (JOB LEVEL), pypa/gh-action-pypi-publish@release/v1 (default index)

Required workflow shape (verify-extras.yml):
  trigger: on.push and on.pull_request (main branch)
  jobs:
    verify:
      strategy.matrix.extra: [memory-pg, memory-chroma, memory-es, dispatch, sandbox, all]
      steps:
        - checkout
        - setup-python 3.12
        - uv build (get a local wheel)
        - python -m venv /tmp/v && source /tmp/v/bin/activate
        - pip install "dist/zeroth_core-*.whl[${{ matrix.extra }}]"
        - run a per-extra import smoke (see import map below)

Per-extra import smoke map (what to import to prove each extra works):
  memory-pg      → python -c "import zeroth.core.memory.pgvector_connector"
  memory-chroma  → python -c "import zeroth.core.memory.chroma_connector"
  memory-es      → python -c "import zeroth.core.memory.es_connector" (or the actual ES backend module — if the file name differs, use the real one; grep src/zeroth/core/memory/ first)
  dispatch       → python -c "import zeroth.core.dispatch.worker"
  sandbox        → python -c "import zeroth.core.sandbox_sidecar"  # empty extra, proves install resolves + module loads
  all            → python -c "import zeroth.core.memory.pgvector_connector, zeroth.core.memory.chroma_connector, zeroth.core.dispatch.worker, zeroth.core.sandbox_sidecar"  # skip es import here because es_connector import is already covered by memory-es entry

Existing CI file (.github/workflows/ci.yml) already installs uv via astral-sh/setup-uv@v5 and python via actions/setup-python@v5 — re-use the same action versions for consistency.
</interfaces>

<manual_user_actions>
The following CANNOT be automated by Claude (no CLI / no API for the PyPI web UI steps). They MUST be completed by the user before the first `gh release create v0.1.1` invocation actually publishes anything. A checkpoint in this plan gates on these:

1. **Reserve + register on pypi.org** (https://pypi.org/manage/account/publishing/):
   - Owner: rrrozhd
   - Repository name: zeroth-core
   - Workflow filename: release-zeroth-core.yml
   - Environment name: pypi
   - PyPI project name: zeroth-core (already reserved — v0.1.0 is live)

2. **Register on test.pypi.org** (https://test.pypi.org/manage/account/publishing/):
   - Owner: rrrozhd
   - Repository name: zeroth-core
   - Workflow filename: release-zeroth-core.yml
   - Environment name: testpypi
   - Note: TestPyPI is a separate index; this registration is REQUIRED for publish-testpypi to succeed. Pitfall #3 in RESEARCH.md.
   - If the zeroth-core name is not yet reserved on test.pypi.org, do a first manual test-upload or use the "pending publisher" flow in the TestPyPI UI.

3. **Create GitHub environments** in the repo settings:
   - https://github.com/rrrozhd/zeroth-core/settings/environments
   - Create environment named `pypi` (no required reviewers, per D-11)
   - Create environment named `testpypi` (no required reviewers)

4. **(Optional) Add ANTHROPIC_API_KEY secret** to the `testpypi` environment (or repo-level secrets) so the smoke-from-testpypi step actually runs `examples/hello.py` against a real LLM. If skipped, the step gracefully prints SKIP and moves on (pitfall #6).
</manual_user_actions>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Author .github/workflows/release-zeroth-core.yml (trusted-publisher release pipeline)</name>
  <files>.github/workflows/release-zeroth-core.yml</files>
  <behavior>
    After this task, `.github/workflows/release-zeroth-core.yml` exists and:
    - Trigger: `on: release: types: [published]` (per D-08)
    - Workflow-level permissions: `contents: read` ONLY (NOT id-token: write — that's job-scoped per D-14 and pitfall #5)
    - Job `build`:
        - runs-on: ubuntu-latest
        - Steps: checkout@v4 → astral-sh/setup-uv@v5 → actions/setup-python@v5 (python 3.12) → assert-tag-matches-version guard → `uv sync --all-groups` → `uv build` → upload dist/ as artifact `dist`
        - Guard script: extract version from pyproject.toml via tomllib, extract tag from `GITHUB_REF_NAME` (strip leading `v`), fail with a clear message on mismatch (per D-07)
    - Job `smoke-install`:
        - needs: [build]
        - Download `dist` artifact → create clean venv → `pip install dist/*.whl` → `python -c "import zeroth.core; print(zeroth.core.__path__)"`
    - Job `test-wheel`:
        - needs: [build]
        - Checkout repo (for the tests/ directory) → download `dist` → clean venv → `pip install dist/*.whl pytest pytest-asyncio` → `pytest tests/ -v --no-header -ra -m "not live"` (respect existing pytest marker convention from pyproject.toml)
        - This runs the test suite against the BUILT WHEEL, not src mode (catches packaging bugs — anti-pattern in RESEARCH.md)
    - Job `publish-testpypi`:
        - needs: [smoke-install, test-wheel]
        - runs-on: ubuntu-latest
        - environment: testpypi
        - permissions: { id-token: write, contents: read }  # JOB-LEVEL per D-14
        - Steps: actions/download-artifact@v4 `dist` → pypa/gh-action-pypi-publish@release/v1 with `repository-url: https://test.pypi.org/legacy/`
        - No explicit `attestations:` flag — default-on in v1.14+ per D-13
    - Job `smoke-from-testpypi`:
        - needs: [publish-testpypi]
        - runs-on: ubuntu-latest
        - env: `ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}`
        - Steps: checkout (for examples/hello.py) → setup-python 3.12 → extract version from pyproject.toml → clean venv → retry-loop pip install from test.pypi.org (5 tries × 15s per pitfall #4) using `--index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ "zeroth-core==$VERSION"` → conditionally run `python examples/hello.py` only if ANTHROPIC_API_KEY is non-empty (pitfall #6)
    - Job `publish-pypi`:
        - needs: [smoke-from-testpypi]
        - runs-on: ubuntu-latest
        - environment: pypi
        - permissions: { id-token: write, contents: read }  # JOB-LEVEL
        - Steps: download `dist` → pypa/gh-action-pypi-publish@release/v1 (default index, attestations default-on)
    - YAML is valid and parseable.
  </behavior>
  <action>
    1. Use the canonical pattern from 28-RESEARCH.md lines 205–307 as the starting point, but adapt per behavior spec above — specifically:
       - Add top-level `permissions: contents: read` (minimum)
       - Job-level `permissions: id-token: write, contents: read` ONLY on publish-testpypi and publish-pypi
       - Add the tag-version assertion step to `build` using tomllib one-liner
       - Use `astral-sh/setup-uv@v5` (matches existing ci.yml)
       - Use `actions/setup-python@v5` with python 3.12
       - Use `actions/upload-artifact@v4` and `actions/download-artifact@v4`
       - test-wheel job must `pytest tests/ -m "not live"` to match pyproject.toml addopts convention
       - smoke-from-testpypi must use the retry loop pattern (5 attempts, 15s sleep between — per pitfall #4)
    2. Ensure all job-level environments (`testpypi`, `pypi`) are spelled exactly as in the manual_user_actions context block — these names must match what the user registers on the PyPI trusted-publisher config.
    3. Do NOT add a schedule trigger or workflow_dispatch — the only trigger is release.published (per D-08).
    4. Do NOT add attestations: true explicitly — it's default-on in v1.14+ and adding the flag pins behavior we don't need.
    5. Validate the YAML syntax:
       `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release-zeroth-core.yml')); print('yaml OK')"`
    6. Sanity-check the key structural assertions via grep/awk (see verify command).
  </action>
  <verify>
    <automated>python3 -c "import yaml; wf=yaml.safe_load(open('.github/workflows/release-zeroth-core.yml')); assert wf.get(True)=={'release': {'types': ['published']}} or wf.get('on')=={'release': {'types': ['published']}}, f'wrong trigger: {wf.get(True) or wf.get(\"on\")}'; jobs=wf['jobs']; assert set(jobs.keys())=={'build','smoke-install','test-wheel','publish-testpypi','smoke-from-testpypi','publish-pypi'}, f'wrong job set: {jobs.keys()}'; assert jobs['publish-testpypi'].get('environment')=='testpypi', f'testpypi env: {jobs[\"publish-testpypi\"].get(\"environment\")}'; assert jobs['publish-pypi'].get('environment')=='pypi', f'pypi env: {jobs[\"publish-pypi\"].get(\"environment\")}'; assert jobs['publish-testpypi']['permissions']=={'id-token':'write','contents':'read'}, f'testpypi perms: {jobs[\"publish-testpypi\"][\"permissions\"]}'; assert jobs['publish-pypi']['permissions']=={'id-token':'write','contents':'read'}, f'pypi perms: {jobs[\"publish-pypi\"][\"permissions\"]}'; assert 'id-token' not in jobs['build'].get('permissions',{}), 'build job must not have id-token'; assert 'id-token' not in jobs['smoke-install'].get('permissions',{}), 'smoke-install must not have id-token'; assert 'id-token' not in jobs['test-wheel'].get('permissions',{}), 'test-wheel must not have id-token'; assert jobs['smoke-install']['needs']==['build'] or jobs['smoke-install']['needs']=='build'; assert 'build' in (jobs['test-wheel']['needs'] if isinstance(jobs['test-wheel']['needs'], list) else [jobs['test-wheel']['needs']]); assert set(jobs['publish-testpypi']['needs'])=={'smoke-install','test-wheel'} if isinstance(jobs['publish-testpypi']['needs'], list) else jobs['publish-testpypi']['needs'] in ('smoke-install','test-wheel'); content=open('.github/workflows/release-zeroth-core.yml').read(); assert 'pypa/gh-action-pypi-publish@release/v1' in content; assert 'test.pypi.org/legacy' in content; assert 'examples/hello.py' in content; assert 'ANTHROPIC_API_KEY' in content; assert 'tomllib' in content, 'tag-version guard missing'; print('OK')"</automated>
  </verify>
  <done>
    `.github/workflows/release-zeroth-core.yml` exists, passes YAML parse, has the six required jobs with correct needs: chain, has job-level id-token: write only on the two publish jobs, uses separate `testpypi` and `pypi` environments, references `examples/hello.py` with an `ANTHROPIC_API_KEY` gate, includes the tag-matches-pyproject-version guard, and uses `pypa/gh-action-pypi-publish@release/v1`.
  </done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Author .github/workflows/verify-extras.yml and update README.md install section</name>
  <files>.github/workflows/verify-extras.yml, README.md</files>
  <behavior>
    After this task:
    - `.github/workflows/verify-extras.yml` exists and:
        - Trigger: `on: [push, pull_request]` (or at minimum on pushes to main + PRs to main)
        - Single job `verify-extras`:
            - runs-on: ubuntu-latest
            - strategy.fail-fast: false (we want to see which extras break)
            - strategy.matrix.extra: [memory-pg, memory-chroma, memory-es, dispatch, sandbox, all]
            - Steps:
                1. checkout@v4
                2. astral-sh/setup-uv@v5
                3. actions/setup-python@v5 (python 3.12)
                4. `uv build` to produce a local wheel
                5. Create a clean venv in `/tmp/extra-venv-${{ matrix.extra }}`
                6. `pip install "dist/zeroth_core-*.whl[${{ matrix.extra }}]"` (using shell-expansion; if shell expansion is awkward, compute the wheel path to a var first)
                7. Run the extra-specific import smoke (per the map in <interfaces> above). Implement as a shell case/dispatch keyed on `${{ matrix.extra }}`.
        - The `sandbox` matrix entry still runs (even though the extra is empty — per D-04 the gate is "install resolves and import works", both true for an empty extra since sandbox_sidecar imports only base deps).
    - Before authoring verify-extras.yml, the executor must `ls src/zeroth/core/memory/` to confirm the actual ES connector filename (could be `es_connector.py`, `elasticsearch_connector.py`, or similar). Use the real filename in the memory-es import step. If no ES connector module exists yet, fall back to `python -c "import elasticsearch; print(elasticsearch.__version__)"` to at least prove the extra resolved.
    - `README.md` has an Install section (or a modification of the existing one) that mentions:
        - Basic install: `pip install zeroth-core`
        - At least one extras install example, e.g.: `pip install "zeroth-core[memory-pg]"` or `pip install "zeroth-core[all]"`
        - A one-line list of available extras
      Per D-24 and the "Deferred: README rewrite" line in CONTEXT.md, this is a SURGICAL edit — do NOT rewrite the full README, do NOT reorganize sections, do NOT add badges or new top-level headings. Add the extras info to the existing install block if one exists; otherwise insert a short "## Install" section after the top-of-file description.
  </behavior>
  <action>
    1. `ls src/zeroth/core/memory/` to find the Elasticsearch connector filename. Record it for use in the memory-es matrix entry. If multiple ES-related files exist, pick the one that actually `import elasticsearch` at the top.
    2. Author `.github/workflows/verify-extras.yml`. Example structure:
       ```yaml
       name: Verify extras
       on:
         push:
           branches: [main]
         pull_request:
           branches: [main]
       jobs:
         verify-extras:
           runs-on: ubuntu-latest
           strategy:
             fail-fast: false
             matrix:
               extra: [memory-pg, memory-chroma, memory-es, dispatch, sandbox, all]
           steps:
             - uses: actions/checkout@v4
             - uses: astral-sh/setup-uv@v5
             - uses: actions/setup-python@v5
               with:
                 python-version: "3.12"
             - name: Build wheel
               run: uv build
             - name: Clean-install extra and import
               shell: bash
               run: |
                 set -euo pipefail
                 WHEEL=$(ls dist/zeroth_core-*.whl)
                 VENV=/tmp/venv-${{ matrix.extra }}
                 python -m venv "$VENV"
                 source "$VENV/bin/activate"
                 pip install --upgrade pip
                 pip install "${WHEEL}[${{ matrix.extra }}]"
                 case "${{ matrix.extra }}" in
                   memory-pg)
                     python -c "import zeroth.core.memory.pgvector_connector; print('memory-pg OK')"
                     ;;
                   memory-chroma)
                     python -c "import zeroth.core.memory.chroma_connector; print('memory-chroma OK')"
                     ;;
                   memory-es)
                     python -c "import zeroth.core.memory.<ACTUAL_ES_MODULE>; print('memory-es OK')"
                     ;;
                   dispatch)
                     python -c "import zeroth.core.dispatch.worker; print('dispatch OK')"
                     ;;
                   sandbox)
                     python -c "import zeroth.core.sandbox_sidecar; print('sandbox OK')"
                     ;;
                   all)
                     python -c "import zeroth.core.memory.pgvector_connector, zeroth.core.memory.chroma_connector, zeroth.core.dispatch.worker, zeroth.core.sandbox_sidecar; print('all OK')"
                     ;;
                 esac
       ```
       Replace `<ACTUAL_ES_MODULE>` with the real filename (minus .py) discovered in step 1.
    3. Update `README.md`:
       - Read current README.md
       - Locate the existing install section (likely near the top — grep for `pip install` or `## Install`)
       - If an install section exists, append the extras examples and the list of available extras (memory-pg, memory-chroma, memory-es, dispatch, sandbox, all) to it
       - If no install section exists, insert a short `## Install` section after the top description block (above the first major section)
       - Keep the addition under ~15 lines. Example content:
         ```markdown
         ## Install

         ```bash
         pip install zeroth-core
         ```

         Optional extras for swappable backends:

         ```bash
         pip install "zeroth-core[memory-pg]"     # Postgres + pgvector memory backend
         pip install "zeroth-core[memory-chroma]" # Chroma memory backend
         pip install "zeroth-core[memory-es]"     # Elasticsearch memory backend
         pip install "zeroth-core[dispatch]"      # Distributed worker (redis + arq)
         pip install "zeroth-core[sandbox]"       # Sandbox sidecar marker
         pip install "zeroth-core[all]"           # Everything above
         ```
         ```
    4. Validate YAML: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/verify-extras.yml'))"`
    5. Do NOT run the workflow locally (no local CI runner assumed) — the gate is YAML validity + structural assertions. The actual matrix runs on the next push.
  </action>
  <verify>
    <automated>python3 -c "import yaml; wf=yaml.safe_load(open('.github/workflows/verify-extras.yml')); jobs=wf['jobs']; j=jobs['verify-extras']; m=j['strategy']['matrix']['extra']; assert set(m)=={'memory-pg','memory-chroma','memory-es','dispatch','sandbox','all'}, f'wrong matrix: {m}'; assert j['strategy']['fail-fast']==False, 'fail-fast should be false'; content=open('.github/workflows/verify-extras.yml').read(); assert 'zeroth.core.memory.pgvector_connector' in content and 'zeroth.core.memory.chroma_connector' in content and 'zeroth.core.dispatch.worker' in content and 'zeroth.core.sandbox_sidecar' in content, 'import smoke steps missing'; readme=open('README.md').read(); assert 'pip install' in readme and 'zeroth-core[' in readme, 'README.md missing extras install examples'; print('OK')"</automated>
  </verify>
  <done>
    `.github/workflows/verify-extras.yml` exists with a 6-entry matrix covering all declared extras, fail-fast disabled, per-extra import smoke steps wired to the real module paths. `README.md` has an install section that includes at least one `pip install "zeroth-core[<extra>]"` example.
  </done>
</task>

<task type="checkpoint:human-action" gate="blocking">
  <name>Task 3: USER ACTION — Register PyPI + TestPyPI trusted publishers and create GitHub environments</name>
  <files>(none — manual PyPI web UI + GitHub repo settings UI)</files>
  <action>Pause execution and present the three steps below to the user. Wait for "registered" confirmation before closing the plan. Do NOT attempt to automate — pypi.org/manage/account/publishing/ and GitHub repo environment settings are web-UI-only.</action>
  <verify>User replies "registered" (or describes blockers). No automated verification available — trusted-publisher registration is stored on pypi.org and test.pypi.org, not in the repo.</verify>
  <done>User confirms: (1) GitHub environments `pypi` and `testpypi` exist in rrrozhd/zeroth-core repo settings, (2) trusted publisher registered on pypi.org for workflow=release-zeroth-core.yml + env=pypi, (3) trusted publisher registered on test.pypi.org for workflow=release-zeroth-core.yml + env=testpypi.</done>
  <what-built>
    The release pipeline workflow (`.github/workflows/release-zeroth-core.yml`) and the continuous extras verification workflow (`.github/workflows/verify-extras.yml`) are now in the repo. The release workflow will fire the next time you create a GitHub Release for the `rrrozhd/zeroth-core` repo — but it cannot actually publish until the PyPI side of trusted-publishing is configured. That configuration is web-UI-only on pypi.org and test.pypi.org and therefore requires manual user action.
  </what-built>
  <how-to-verify>
    Complete the following, in order. Claude pauses until you confirm.

    **Step 1 — Create GitHub environments** in the repo settings:
    - Go to https://github.com/rrrozhd/zeroth-core/settings/environments
    - Click "New environment", name: `pypi`, save (no required reviewers, leave defaults)
    - Click "New environment" again, name: `testpypi`, save
    - (Optional) add `ANTHROPIC_API_KEY` as a secret on the `testpypi` environment if you want the `smoke-from-testpypi` job to actually run `examples/hello.py` against a real LLM. Skipping this is fine — the workflow gracefully prints SKIP.

    **Step 2 — Register the trusted publisher on pypi.org**:
    - Go to https://pypi.org/manage/account/publishing/
    - Under "Add a new pending publisher" (or on the zeroth-core project page if you're logged in as an owner), fill in:
        - PyPI Project Name: `zeroth-core`
        - Owner: `rrrozhd`
        - Repository name: `zeroth-core`
        - Workflow name: `release-zeroth-core.yml`
        - Environment name: `pypi`
    - Click "Add"

    **Step 3 — Register the trusted publisher on test.pypi.org** (SEPARATE registration — per pitfall #3, TestPyPI is a different index):
    - Go to https://test.pypi.org/manage/account/publishing/
    - Fill in:
        - PyPI Project Name: `zeroth-core`
        - Owner: `rrrozhd`
        - Repository name: `zeroth-core`
        - Workflow name: `release-zeroth-core.yml`
        - Environment name: `testpypi`
    - Click "Add"
    - If TestPyPI says the name is not owned, use the "pending publisher" flow — it allows first-upload to claim the name.

    **Step 4 — Confirm to Claude**:
    Type "registered" once all three steps are done, or describe any errors you encountered.

    (Do NOT create a GitHub Release for v0.1.1 yet — that's the post-phase follow-up step. This checkpoint only covers the publisher registration so the workflow is READY to run.)
  </how-to-verify>
  <resume-signal>Type "registered" when all three steps are complete, or describe issues.</resume-signal>
</task>

</tasks>

<verification>
Overall plan verification:

1. `.github/workflows/release-zeroth-core.yml` exists, YAML valid, six required jobs with correct structure, separate testpypi/pypi environments, job-level id-token permissions, tag-version guard, examples/hello.py reference with ANTHROPIC_API_KEY gate
2. `.github/workflows/verify-extras.yml` exists, YAML valid, six-entry matrix, fail-fast false, per-extra import smoke steps
3. `README.md` install section mentions extras
4. User confirms trusted publishers are registered on both pypi.org and testpypi.org and `pypi` / `testpypi` GitHub environments exist
5. No workflow actually runs yet — the first release is a post-phase follow-up step (creating `v0.1.1` GitHub Release), documented in the phase SUMMARY and not gated by this plan
</verification>

<success_criteria>
- [ ] `.github/workflows/release-zeroth-core.yml` is committed with the six-stage trusted-publisher pipeline (per D-08/D-09/D-10/D-11/D-12/D-13/D-14)
- [ ] Release workflow's publish jobs are scoped to `testpypi` and `pypi` GitHub environments respectively (per D-11 + research-resolved separate envs)
- [ ] Release workflow has `id-token: write` at JOB level only, not workflow level (per D-14 / pitfall #5)
- [ ] Release workflow uses `pypa/gh-action-pypi-publish@release/v1` with default-on Sigstore attestations (per D-13)
- [ ] Release workflow has tag-matches-version guard in the build job (per D-07)
- [ ] Release workflow's test-wheel job runs pytest against the built wheel, not src mode (anti-pattern in RESEARCH.md)
- [ ] Release workflow's smoke-from-testpypi job retries pip install to handle TestPyPI indexing lag (pitfall #4)
- [ ] Release workflow's smoke-from-testpypi job runs `examples/hello.py` with ANTHROPIC_API_KEY gate (per D-18/D-19 / pitfall #6 / PKG-06)
- [ ] `.github/workflows/verify-extras.yml` is committed with a 6-entry matrix covering all declared extras and per-extra import smokes (per D-04 / PKG-03)
- [ ] `README.md` install section mentions at least one extras example (per D-24)
- [ ] User has registered trusted publishers on pypi.org AND test.pypi.org and created the `pypi` / `testpypi` GitHub environments (per manual_user_actions)
- [ ] Phase 28 post-phase follow-up is documented in the phase SUMMARY: "Create GitHub Release v0.1.1 to trigger the first trusted-publisher release of zeroth-core==0.1.1"
</success_criteria>

<output>
After completion, create `.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-03-SUMMARY.md` covering: the two workflow files with their job structures, the README.md install-section diff, the user-action checkpoint outcome (confirmation text), and the explicit post-phase follow-up item ("Operator creates GitHub Release v0.1.1 to trigger first trusted-publisher run; PKG-02/PKG-05/PKG-06 close when that run succeeds end-to-end").
</output>
