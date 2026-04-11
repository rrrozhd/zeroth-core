---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 05
type: execute
wave: 3
depends_on: [02, 03, 04]
files_modified:
  - .github/workflows/docs.yml
  - README.md
  - tests/test_docs_phase30.py
autonomous: false
requirements:
  - SITE-02
  - SITE-01
  - SITE-04
tags: [docs, ci, github-pages, deploy]
user_setup:
  - service: github-pages
    why: "First-time Pages source must be set manually after the first successful gh-deploy push"
    dashboard_config:
      - task: "Enable GitHub Pages on the zeroth-core repo after the first green main-branch run"
        location: "https://github.com/rrrozhd/zeroth-core/settings/pages — set Source to 'Deploy from a branch', branch `gh-pages`, folder `/ (root)`, save"
must_haves:
  truths:
    - "`.github/workflows/docs.yml` exists, is valid YAML, and triggers on `push: branches: [main]` and `pull_request: branches: [main]`"
    - "Workflow has `permissions: contents: write` at the job or workflow level (required for gh-deploy to push to gh-pages)"
    - "The build step runs `uv run mkdocs build --strict` on every trigger (PR + push)"
    - "The deploy step runs `uv run mkdocs gh-deploy --force --no-history` and is gated by `if: github.event_name == 'push' && github.ref == 'refs/heads/main'` (so forked PRs don't 403)"
    - "`README.md` has a top-of-file link to the live docs URL `https://rrrozhd.github.io/zeroth/`"
    - "A final `mkdocs build --strict` from the repo root produces zero warnings and the resulting site contains all 4 Diátaxis sections and all tutorial pages"
    - "PHASE-LEVEL GAP recorded in SUMMARY.md: SITE-03 (PR previews) is deferred — noted with rationale in the phase SUMMARY and in STATE.md blockers/concerns"
  artifacts:
    - path: ".github/workflows/docs.yml"
      provides: "Build-on-PR + deploy-on-main workflow using mkdocs gh-deploy"
    - path: "README.md"
      provides: "Updated docs link at the top (below existing Install section or above it)"
  key_links:
    - from: ".github/workflows/docs.yml"
      to: "mkdocs.yml"
      via: "uv run mkdocs build --strict"
      pattern: "mkdocs build.*--strict"
    - from: ".github/workflows/docs.yml"
      to: "gh-pages branch"
      via: "mkdocs gh-deploy --force"
      pattern: "gh-deploy.*--force"
    - from: "README.md"
      to: "https://rrrozhd.github.io/zeroth/"
      via: "markdown link at top"
      pattern: "rrrozhd\\.github\\.io/zeroth"
---

<objective>
Wire up the docs deploy pipeline: a GitHub Actions workflow that
strict-builds on every PR and deploys to `gh-pages` on every push to
main, update the README with a link to the live site, and execute a
final phase-gate validation that the entire site builds cleanly end to
end. Record SITE-03 (PR previews) deferral explicitly.

Purpose: Satisfy SITE-02 (deploy on every main commit) and finalize
SITE-01/SITE-04 verification now that all content (plans 02-04) is in
place. Include the one human-required step to enable GitHub Pages.

Output: docs.yml workflow + README update + user-action checkpoint +
deferred-requirement documentation.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-CONTEXT.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-RESEARCH.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-02-SUMMARY.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-03-SUMMARY.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-04-SUMMARY.md
@mkdocs.yml
@README.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write .github/workflows/docs.yml + update README + shape tests</name>
  <files>
    .github/workflows/docs.yml
    README.md
    tests/test_docs_phase30.py
  </files>
  <action>
    1. Create `.github/workflows/docs.yml` using the exact workflow
       from 30-RESEARCH.md "Pattern 2":

       ```yaml
       name: docs
       on:
         push:
           branches: [main]
         pull_request:
           branches: [main]

       permissions:
         contents: write

       jobs:
         build:
           runs-on: ubuntu-latest
           steps:
             - uses: actions/checkout@v4
               with:
                 fetch-depth: 0
             - uses: astral-sh/setup-uv@v5
             - uses: actions/setup-python@v5
               with:
                 python-version: "3.12"
             - run: uv sync --extra docs
             - name: Build site (strict)
               run: uv run mkdocs build --strict
             - name: Deploy to GitHub Pages
               if: github.event_name == 'push' && github.ref == 'refs/heads/main'
               run: uv run mkdocs gh-deploy --force --no-history
       ```

       Confirm the action versions against `.github/workflows/ci.yml` and
       other existing workflows — if the repo pins different versions,
       match them for consistency.

    2. Update `README.md` — add a prominent link to the live docs near
       the top. A good spot is a new "Documentation" section right
       after the title tagline, before "Install":

       ```markdown
       ## Documentation

       Full documentation lives at **<https://rrrozhd.github.io/zeroth/>** —
       start with the [Getting Started tutorial](https://rrrozhd.github.io/zeroth/tutorials/getting-started/)
       or the [Governance Walkthrough](https://rrrozhd.github.io/zeroth/tutorials/governance-walkthrough/).
       ```

       Do NOT break the existing "Install" / "Why Zeroth" flow. Add,
       don't rearrange.

    3. Extend `tests/test_docs_phase30.py` with:
         * `test_docs_workflow_exists_and_valid_yaml`: assert file
           exists, yaml.safe_load succeeds, has `name: docs`, both
           triggers present.
         * `test_docs_workflow_build_is_strict`: assert the workflow
           string contains `mkdocs build --strict`.
         * `test_docs_workflow_deploy_is_main_only`: assert the deploy
           step has `if:` condition containing `github.event_name ==
           'push'` and `refs/heads/main`.
         * `test_docs_workflow_has_contents_write_permission`: assert
           `permissions: contents: write` is set.
         * `test_readme_links_to_live_docs`: assert `README.md`
           contains `https://rrrozhd.github.io/zeroth/`.

    4. Run `uv run pytest tests/test_docs_phase30.py -v` — all green.
    5. `uv run ruff check tests/test_docs_phase30.py` and format.
  </action>
  <verify>
    <automated>uv run pytest tests/test_docs_phase30.py -v &amp;&amp; uv run ruff check tests/test_docs_phase30.py</automated>
  </verify>
  <done>
    - docs.yml exists, strict-builds on PR, deploys to gh-pages on main only, has contents:write
    - README has a Documentation section linking to the live URL
    - Five new shape tests green
  </done>
</task>

<task type="auto">
  <name>Task 2: Final phase-gate validation (strict build + full test suite + deferral doc)</name>
  <files>
    .planning/STATE.md
  </files>
  <action>
    1. Run the complete phase gate:
         - `uv run mkdocs build --strict` — MUST be green with zero warnings
         - `uv run pytest -v` — full suite, no new regressions vs baseline
         - `uv run ruff check src/ tests/ examples/` — clean
         - For each example with SKIP guard: confirm exit 0 without
           OPENAI_API_KEY set:
             `OPENAI_API_KEY= uv run python examples/hello.py`
             `OPENAI_API_KEY= uv run python examples/first_graph.py`
             `OPENAI_API_KEY= uv run python examples/approval_demo.py`
             `OPENAI_API_KEY= uv run python examples/governance_walkthrough.py`

    2. Inspect `site/` directory produced by the strict build:
         - `site/sitemap.xml` should list all tutorial pages
         - `site/index.html` should exist
         - `site/tutorials/getting-started/02-first-graph/index.html` should exist
         - `site/tutorials/governance-walkthrough/index.html` should exist

    3. Update `.planning/STATE.md` Blockers/Concerns section:
       Add a bullet:
       `- SITE-03 (PR preview deploys) deferred to follow-up phase — GitHub Pages does not natively support PR previews and the user accepted deferral during Phase 30 planning. Re-open if/when docs move to Cloudflare Pages or Netlify.`
       Also update the STATE.md `stopped_at` and `last_activity` to
       reflect Phase 30 plan 05 completion expectations; leave phase
       completion for the orchestrator to commit.

    4. Document the deferred requirement in the forthcoming phase
       SUMMARY.md (plan-05-SUMMARY.md will note SITE-03 as the single
       gap).
  </action>
  <verify>
    <automated>uv run mkdocs build --strict &amp;&amp; uv run pytest -q &amp;&amp; OPENAI_API_KEY= uv run python examples/hello.py &amp;&amp; OPENAI_API_KEY= uv run python examples/first_graph.py &amp;&amp; OPENAI_API_KEY= uv run python examples/approval_demo.py &amp;&amp; OPENAI_API_KEY= uv run python examples/governance_walkthrough.py</automated>
  </verify>
  <done>
    - Strict mkdocs build green
    - Full pytest suite green (no regressions)
    - All four examples SKIP cleanly with exit 0
    - site/ tree contains all expected pages + sitemap
    - STATE.md has SITE-03 deferral entry
  </done>
</task>

<task type="checkpoint:human-action" gate="blocking">
  <name>Task 3 (checkpoint): Enable GitHub Pages on rrrozhd/zeroth-core</name>
  <files>(none — this is a one-time manual setup in the GitHub web UI; no repo files change)</files>
  <action>Pause execution and prompt the user to perform the GitHub Pages first-time source selection described under &lt;how-to-verify&gt; below. This is the only step in Phase 30 Claude cannot automate — the gh-pages source dropdown on a repo that has never deployed Pages before requires a human click-through. Do not proceed until the user replies per &lt;resume-signal&gt;.</action>
  <verify><automated>echo "Manual checkpoint — user must confirm live docs URL serves the landing page (see how-to-verify below)."</automated></verify>
  <done>User has replied "approved" AND https://rrrozhd.github.io/zeroth/ serves the Zeroth landing page with all four Diátaxis sections visible, the Getting Started tutorial pages rendering, the Governance Walkthrough page rendering, and the built-in search working.</done>
  <what-built>
    - `.github/workflows/docs.yml` that builds strict-mode on every PR and runs `mkdocs gh-deploy` on push to main
    - Full mkdocs-material site with Diátaxis IA, Getting Started tutorial, and Governance Walkthrough
    - README link to the live docs URL
  </what-built>
  <how-to-verify>
    GitHub Pages must be enabled manually — there is no CLI/API path for
    first-time Pages source selection on a repo that has never deployed
    Pages before.

    1. Merge this PR / push plan-05 commits to `main`.
    2. Wait for the `docs` workflow's first green run on `main`. This
       run creates the `gh-pages` branch via `mkdocs gh-deploy --force`.
    3. Go to https://github.com/rrrozhd/zeroth-core/settings/pages
    4. Under "Build and deployment" → "Source", choose
       **"Deploy from a branch"**.
    5. Under "Branch", choose **`gh-pages`** and folder **`/ (root)`**.
    6. Click **Save**.
    7. Wait 1-2 minutes for GitHub to provision the site.
    8. Visit **https://rrrozhd.github.io/zeroth/** — you should see the
       Zeroth landing page with the Choose Your Path tabs and the
       hello.py snippet embed.
    9. Click through to Tutorials → Getting Started → First graph.
       Verify the embedded `first_graph.py` snippet renders with syntax
       highlighting.
    10. Click through to Tutorials → Governance Walkthrough. Verify the
        three scenario sections render and the bottom full-file embed
        shows the example.
    11. Try the built-in search: click the search icon (or press `/`),
        type "approval", expect hits in the tutorial pages.

    Common issues:
    - **404 on rrrozhd.github.io/zeroth/**: Pages source not saved, or
      the `gh-pages` branch does not yet exist. Re-run the docs workflow
      on main and retry.
    - **404 on /tutorials/...**: `site_url` mismatch. If the repo is
      actually named `zeroth-core` and Pages publishes at
      `rrrozhd.github.io/zeroth-core/`, then `mkdocs.yml`'s `site_url`
      + the README link both need to be fixed. Report which one is
      correct.
  </how-to-verify>
  <resume-signal>
    Reply with:
    - "approved" if the live site works end to end
    - "site_url wrong, should be rrrozhd.github.io/zeroth-core/" if the
      repo slug differs — the executor will fix mkdocs.yml + README and
      re-run
    - Describe any other issue and the executor will diagnose
  </resume-signal>
</task>

</tasks>

<verification>
- `uv run mkdocs build --strict` → green
- `uv run pytest -q` → green (no regressions)
- `uv run pytest tests/test_docs_phase30.py -v` → all green
- All example SKIP paths exit 0
- Human verification: live docs URL returns the Zeroth landing page
</verification>

<success_criteria>
- SITE-02 shipped: GHA workflow builds on PR and deploys to gh-pages on main
- SITE-01 + SITE-04 verified end-to-end through the final strict build
- Live docs URL serves the landing page, Getting Started tutorial, and Governance Walkthrough
- README links to the live docs
- SITE-03 (PR previews) explicitly deferred with rationale recorded in STATE.md and SUMMARY.md
- Human checkpoint completes after manual Pages source selection
</success_criteria>

<output>
After completion, create `.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-05-SUMMARY.md` documenting: the final workflow file, the live docs URL confirmed working, and an explicit "Deferred Requirements" section listing SITE-03 (PR previews) with rationale "GitHub Pages has no native PR preview; user-accepted deferral per Phase 30 CONTEXT D-06. Re-open if docs deploy migrates to Cloudflare Pages or Netlify."
</output>
