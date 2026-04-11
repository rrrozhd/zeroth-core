---
phase: 28
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - LICENSE
  - CHANGELOG.md
  - CONTRIBUTING.md
  - examples/hello.py
  - .planning/STATE.md
autonomous: true
requirements:
  - PKG-04
  - PKG-06
tags:
  - oss-metadata
  - license
  - changelog
  - contributing
  - examples

must_haves:
  truths:
    - "Repo root has LICENSE (full canonical Apache-2.0 text)"
    - "Repo root has CHANGELOG.md in keepachangelog 1.1.0 format with a seeded [0.1.1] entry"
    - "Repo root has CONTRIBUTING.md covering dev setup, PR conventions, issues, and LICENSE link"
    - "examples/hello.py exists and runs end-to-end against a real LLM when ANTHROPIC_API_KEY is set"
    - "examples/hello.py prints a clear skip/error message when ANTHROPIC_API_KEY is absent, exit code 0"
    - "STATE.md blockers list is reconciled: stale Regulus entries removed, remaining manual PyPI trusted-publisher user actions recorded"
  artifacts:
    - path: "LICENSE"
      provides: "Apache-2.0 canonical license text"
      min_lines: 175
      contains: "Apache License"
    - path: "CHANGELOG.md"
      provides: "keepachangelog 1.1.0 format with seeded 0.1.1 entry"
      contains: "Keep a Changelog"
    - path: "CONTRIBUTING.md"
      provides: "Dev setup, PR conventions, issues, LICENSE link"
      contains: "uv sync"
    - path: "examples/hello.py"
      provides: "PKG-06 acceptance fixture — clean-venv-installable program that exercises zeroth.core end-to-end"
      min_lines: 20
      contains: "zeroth.core"
    - path: ".planning/STATE.md"
      provides: "Reconciled blockers/concerns list per D-25"
  key_links:
    - from: "CONTRIBUTING.md"
      to: "LICENSE"
      via: "markdown link"
      pattern: "\\[LICENSE\\]\\(LICENSE\\)|LICENSE file"
    - from: "examples/hello.py"
      to: "zeroth.core"
      via: "import statement"
      pattern: "import zeroth\\.core|from zeroth\\.core"
    - from: "CHANGELOG.md [0.1.1] section"
      to: "PyPI release"
      via: "version number match with pyproject.toml"
      pattern: "\\[0\\.1\\.1\\]"
---

<objective>
Add the three OSS-grade repo metadata files required by PKG-04 (LICENSE, CHANGELOG.md, CONTRIBUTING.md), ship the canonical `examples/hello.py` fixture that satisfies PKG-06, and reconcile the stale Regulus blockers in STATE.md per D-25.

Purpose: A PyPI release for a public library must have a license file, an auditable changelog, and a contributor on-ramp. PKG-06 requires a clean-venv installable script that proves the library works end-to-end. Plan 03's release workflow will execute `examples/hello.py` as its final smoke test, so this plan must land the file before Plan 03 runs.

Output: Four new files at the repo root (LICENSE, CHANGELOG.md, CONTRIBUTING.md, examples/hello.py) plus a surgical edit to `.planning/STATE.md`. No source code changes.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-CONTEXT.md
@.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-RESEARCH.md
@.planning/STATE.md
@README.md
@CLAUDE.md

<interfaces>
<!-- What examples/hello.py needs to import. Discovered from Phase 27 runtime. -->
<!-- Executor: if the exact graph API differs, follow existing test patterns in tests/ — -->
<!-- the goal is a tiny working graph, NOT a perfect API showcase. -->

Public surface likely usable for a minimal hello-world:
  - zeroth.core.graph — graph construction primitives
  - zeroth.core.orchestrator — run a graph
  - zeroth.core.agent_runtime — agent node
  - zeroth.core.contracts — node/edge types

litellm is a base dep, so `from litellm import completion` or equivalent works without an extra.
ANTHROPIC_API_KEY is the recommended gate (per D-19); executor may also accept OPENAI_API_KEY as a fallback if the graph builder's agent config supports provider=anthropic naturally.

If the full orchestrator requires an extra (memory, dispatch, etc.), fall back to a truly minimal version: one agent node, no memory, no dispatch, synchronous run. This is a "does the import graph load and does the LLM respond" smoke test, not a feature tour.
</interfaces>

<stale_blockers_to_remove>
Current `.planning/STATE.md` ### Blockers/Concerns section contains three entries. Per resolved context + D-25:
  - REMOVE: "Regulus has no GitHub remote yet — blocks publishing `econ-instrumentation-sdk` to PyPI, which blocks a clean `zeroth-core` dependency declaration" (stale — econ-sdk 0.1.1 is on PyPI, pinned at commit 78c2076, PKG-01 satisfied)
  - KEEP but REWORD: "PyPI trusted publisher setup for `zeroth-core` ... requires manual user action on pypi.org" — reword to drop `econ-instrumentation-sdk` (out of scope per D-10)
  - KEEP as-is: "Local parent directory /Users/dondoe/coding/zeroth/ needs to be renamed to zeroth-archive/ ..." — unrelated to Phase 28 scope, leave

Also update ### Pending Todos:
  - REMOVE: "Configure the missing Regulus GitHub remote so the econ SDK can be published from a normal public origin" (out of scope, not gating Phase 28)
  - KEEP: "Plan and execute Phase 28 publication work" — will be ticked off when Phase 28 completes
  - REPLACE: "Complete the manual PyPI trusted-publisher setup for both packages" → "Complete the manual PyPI trusted-publisher setup for zeroth-core on pypi.org AND test.pypi.org (two separate registrations)"
</stale_blockers_to_remove>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Add LICENSE, CHANGELOG.md, and CONTRIBUTING.md at repo root</name>
  <files>LICENSE, CHANGELOG.md, CONTRIBUTING.md</files>
  <behavior>
    After this task:
    - `LICENSE` contains the full canonical Apache-2.0 text verbatim from https://www.apache.org/licenses/LICENSE-2.0.txt (per D-15). Starts with "Apache License / Version 2.0, January 2004". No copyright line substitutions beyond the standard "Copyright [yyyy] [name of copyright owner]" — per D-15 this is the full canonical text, not a filled-in template. File is ~175–205 lines.
    - `CHANGELOG.md` is in keepachangelog 1.1.0 format (per D-16) with:
        - Header block: "# Changelog\n\nAll notable changes to this project will be documented in this file.\n\nThe format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),\nand this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).\n"
        - `## [Unreleased]` section (empty, placeholder)
        - `## [0.1.1] - 2026-04-11` section with subsections summarizing: "### Added" (Apache-2.0 license, keepachangelog, contributing guide, py.typed marker, [project.urls], classifiers, examples/hello.py, optional extras memory-pg/memory-chroma/memory-es/dispatch/sandbox/all, trusted-publisher release workflow, Sigstore attestations), "### Changed" (dependencies carved into base + extras; hatchling bumped to >=1.27 for PEP 639), "### Fixed" (none yet — leave absent or note "n/a for first public release")
        - NO retrospective history — this is the first public changelog (per D-16)
    - `CONTRIBUTING.md` is ~1 page (per D-17) covering, in this order, as markdown sections:
        1. "## Development setup" — `git clone`, `uv sync --all-extras --all-groups`, `uv run pytest -v`, `uv run ruff check src tests`, `uv run ruff format src` (per CLAUDE.md Build & Test)
        2. "## Running the example" — `python examples/hello.py` with a note about ANTHROPIC_API_KEY
        3. "## Pull request conventions" — commit format (`type(scope): subject`, e.g., `feat(memory): add redis backend`), branch naming (`feat/...`, `fix/...`, `docs/...`), link PRs to issues when applicable
        4. "## Filing issues" — use GitHub Issues at https://github.com/rrrozhd/zeroth-core/issues, include repro steps, zeroth-core version, Python version
        5. "## License" — link to `LICENSE` file, one-sentence summary "By contributing you agree your contributions will be licensed under Apache-2.0"
        6. "## Code of conduct" — short placeholder: "A community code of conduct will be added in a future phase; for now, please communicate professionally and in good faith."
    - All three files use UTF-8, LF line endings, no trailing whitespace.
  </behavior>
  <action>
    1. **LICENSE**: Write the canonical Apache-2.0 text. The exact content is available at https://www.apache.org/licenses/LICENSE-2.0.txt — the standard version that begins "Apache License / Version 2.0, January 2004 / http://www.apache.org/licenses/" and ends with "END OF TERMS AND CONDITIONS" followed by the APPENDIX. Include the APPENDIX block verbatim. Do NOT fill in the copyright holder name/year in the APPENDIX — leave the `[yyyy]` and `[name of copyright owner]` placeholders since that's how the canonical template ships, and the project-level copyright attribution belongs in NOTICE (not required for Phase 28).
    2. **CHANGELOG.md**: Write per the spec above. Use today's date 2026-04-11 for the [0.1.1] section. Keep entries concrete and map them to Phase 28 deliverables. Example shape:
       ```
       # Changelog

       All notable changes to this project will be documented in this file.

       The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
       and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

       ## [Unreleased]

       ## [0.1.1] - 2026-04-11

       ### Added
       - Apache-2.0 `LICENSE` file at repo root.
       - `CHANGELOG.md` in keepachangelog 1.1.0 format.
       - `CONTRIBUTING.md` with dev setup, PR conventions, and issue filing guide.
       - PEP 561 `py.typed` marker — downstream users now receive type hints.
       - `[project.urls]` block (Homepage, Source, Issues, Changelog).
       - PyPI classifiers and keywords for searchability.
       - `examples/hello.py` — minimal runnable fixture (PKG-06).
       - Optional extras: `[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]` (PKG-03).
       - GitHub Actions trusted-publisher release workflow with TestPyPI staging and Sigstore attestations (PKG-05).

       ### Changed
       - Dependencies carved into a minimal base + six optional extras. Installing bare `zeroth-core` no longer pulls `psycopg`, `pgvector`, `chromadb-client`, `elasticsearch`, `redis`, or `arq`.
       - Build backend bumped to `hatchling>=1.27` for PEP 639 SPDX license-expression support.
       ```
       (Executor can embellish but must keep the [0.1.1] date and version.)
    3. **CONTRIBUTING.md**: Write per the spec above. Keep it tight — roughly 50–120 lines total. Link to `LICENSE` using a relative markdown link `[LICENSE](LICENSE)`. Do NOT add RFC, governance, or code-of-conduct sections beyond the placeholder.
    4. Verify files are valid UTF-8 and render: `python3 -c "open('LICENSE').read(); open('CHANGELOG.md').read(); open('CONTRIBUTING.md').read(); print('all readable')"`.
  </action>
  <verify>
    <automated>python3 -c "import os; assert os.path.exists('LICENSE') and os.path.exists('CHANGELOG.md') and os.path.exists('CONTRIBUTING.md'); lic=open('LICENSE').read(); assert 'Apache License' in lic and 'Version 2.0, January 2004' in lic and 'END OF TERMS AND CONDITIONS' in lic and 'APPENDIX' in lic, 'LICENSE not canonical Apache-2.0'; assert len(lic.splitlines())>150, f'LICENSE too short: {len(lic.splitlines())} lines'; cl=open('CHANGELOG.md').read(); assert 'Keep a Changelog' in cl and 'keepachangelog.com/en/1.1.0' in cl and '[0.1.1]' in cl and '2026-04-11' in cl and '### Added' in cl and '### Changed' in cl and '[Unreleased]' in cl, 'CHANGELOG.md missing required keepachangelog elements'; cn=open('CONTRIBUTING.md').read(); assert 'uv sync' in cn and 'Apache-2.0' in cn and 'LICENSE' in cn and 'pull request' in cn.lower() and 'examples/hello.py' in cn, 'CONTRIBUTING.md missing required sections'; print('OK')"</automated>
  </verify>
  <done>
    `LICENSE`, `CHANGELOG.md`, `CONTRIBUTING.md` exist at repo root with the content specified; the verification script prints "OK" confirming canonical Apache-2.0 text, keepachangelog format with [0.1.1] entry, and CONTRIBUTING.md with required sections.
  </done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Write examples/hello.py and reconcile STATE.md blockers</name>
  <files>examples/hello.py, .planning/STATE.md</files>
  <behavior>
    After this task:
    - `examples/hello.py` is a single-file Python program, ~30–80 lines, that:
        - Has a module docstring explaining it's the minimal PKG-06 fixture.
        - Imports `zeroth.core` (and a narrowly-scoped subset — graph/orchestrator/agent/contracts as needed for the minimal graph).
        - Reads `ANTHROPIC_API_KEY` from env (per D-19). If absent, prints a clear message to stderr: `"SKIP: set ANTHROPIC_API_KEY to run examples/hello.py against a real LLM"` and exits with code 0 (so CI fork-PRs don't break — pitfall #6).
        - If the key is present, builds the smallest possible graph that exercises agent runtime + LLM call + orchestrator run, runs it, prints the result to stdout, and exits with code 0.
        - If the graph builder for a full agent run is too complex to fit in a short example (e.g., requires config files, service bootstrap), fall back to an even simpler "tour" that imports `zeroth.core`, prints `zeroth.core.__path__`, calls `litellm.completion(model="anthropic/claude-3-haiku-20240307", messages=[{"role":"user","content":"Say hello in 5 words."}])` directly (litellm is a base dep), and prints the response. Document the fallback in a comment referencing "Phase 30 will replace this with a full graph walkthrough."
        - NO hardcoded API keys. NO print-and-crash — always exit code 0 when the key is missing.
    - File is committed under `examples/hello.py`. The `examples/` directory is created if it doesn't exist (it doesn't currently).
    - `.planning/STATE.md` is surgically updated (see stale_blockers_to_remove block in context):
        - `### Blockers/Concerns` loses the Regulus entry; the PyPI trusted-publisher entry is reworded to zeroth-core-only.
        - `### Pending Todos` loses the Regulus remote entry; the trusted-publisher todo is expanded to name pypi.org AND test.pypi.org.
        - `last_updated` frontmatter field is bumped to `"2026-04-11T00:00:00Z"` (or current UTC).
        - NO other section is touched — this is a surgical edit, not a rewrite.
  </behavior>
  <action>
    1. Create the `examples/` directory (Write tool creates parents).
    2. Write `examples/hello.py`. Prefer a real zeroth.core graph if the API surface is ergonomic enough for ~30 lines. If not, use the litellm fallback pattern described in behavior. Required skeleton:
       ```python
       """Minimal PKG-06 acceptance fixture for zeroth-core.

       Run: python examples/hello.py
       Requires: ANTHROPIC_API_KEY env var (otherwise prints SKIP and exits 0).

       Phase 30 will wrap this file in a Getting Started tutorial.
       """

       import os
       import sys


       def main() -> int:
           if not os.environ.get("ANTHROPIC_API_KEY"):
               print(
                   "SKIP: set ANTHROPIC_API_KEY to run examples/hello.py against a real LLM",
                   file=sys.stderr,
               )
               return 0

           import zeroth.core  # noqa: F401 — import-smoke for PKG-06
           # ... minimal graph or litellm direct call ...
           # Print the LLM response.
           return 0


       if __name__ == "__main__":
           raise SystemExit(main())
       ```
       Executor fills in the body. If using litellm direct call, use `anthropic/claude-3-haiku-20240307` (cheap, fast) with a short prompt like "Say hello from zeroth-core in one sentence." Print the completion content to stdout.
    3. Run the example WITHOUT a key to verify the skip path:
       `ANTHROPIC_API_KEY= uv run python examples/hello.py` — must print SKIP line to stderr and exit 0.
    4. If an ANTHROPIC_API_KEY is available in the developer's env, also run it with the key and confirm it produces real output. This is OPTIONAL (the automated verify only asserts the skip path; full-path is covered by the CI job in Plan 03).
    5. Surgically edit `.planning/STATE.md`:
       - In `### Blockers/Concerns`:
         - Delete the bullet starting "Regulus has no GitHub remote yet..."
         - Replace "PyPI trusted publisher setup for `zeroth-core` and `econ-instrumentation-sdk` ..." with:
           "PyPI trusted-publisher setup for `zeroth-core` requires manual user action — must register the publisher on pypi.org (environment `pypi`) AND test.pypi.org (environment `testpypi`) separately. econ-instrumentation-sdk publishing lives in the Regulus repo and is out of scope for Phase 28."
         - Keep the directory-rename bullet untouched.
       - In `### Pending Todos`:
         - Delete "Configure the missing Regulus GitHub remote ..."
         - Replace "Complete the manual PyPI trusted-publisher setup for both packages" with "Complete the manual PyPI trusted-publisher setup for zeroth-core on pypi.org AND test.pypi.org (two separate registrations)"
       - Bump `last_updated` in YAML frontmatter to `"2026-04-11T00:00:00Z"`.
       - Do NOT modify `## Current Position`, `## Performance Metrics`, `### Decisions`, `## Session Continuity`, or any other section.
    6. Do NOT run `git commit` — that's handled at phase level.
  </action>
  <verify>
    <automated>python3 -c "import os, subprocess; assert os.path.exists('examples/hello.py'); src=open('examples/hello.py').read(); assert 'ANTHROPIC_API_KEY' in src and 'zeroth.core' in src and 'SKIP' in src, 'hello.py missing required elements'; r=subprocess.run(['python3','examples/hello.py'], env={**os.environ, 'ANTHROPIC_API_KEY': ''}, capture_output=True, text=True); assert r.returncode==0, f'hello.py skip path exit={r.returncode} stderr={r.stderr}'; assert 'SKIP' in r.stderr, f'expected SKIP in stderr, got: {r.stderr}'; st=open('.planning/STATE.md').read(); assert 'Regulus has no GitHub remote' not in st, 'stale Regulus blocker still present'; assert 'test.pypi.org' in st, 'reconciled trusted-publisher entry missing test.pypi.org mention'; assert '2026-04-11' in st, 'last_updated not bumped'; print('OK')"</automated>
  </verify>
  <done>
    `examples/hello.py` exists, runs cleanly with no key (prints SKIP to stderr, exit 0), imports `zeroth.core`. `.planning/STATE.md` has the Regulus blocker removed, the trusted-publisher entry reworded to zeroth-core-only with pypi.org AND test.pypi.org explicitly named, and `last_updated` bumped.
  </done>
</task>

</tasks>

<verification>
Overall plan verification:

1. `ls LICENSE CHANGELOG.md CONTRIBUTING.md examples/hello.py` — all four files exist
2. Task 1 verify command passes (LICENSE canonical, CHANGELOG keepachangelog + [0.1.1], CONTRIBUTING sections)
3. Task 2 verify command passes (hello.py skip path works, STATE.md reconciled)
4. `ANTHROPIC_API_KEY= python3 examples/hello.py` exits 0 with SKIP message
5. `grep -c "Regulus has no GitHub remote" .planning/STATE.md` returns 0
</verification>

<success_criteria>
- [ ] `LICENSE` at repo root contains canonical Apache-2.0 text (per D-15 / PKG-04)
- [ ] `CHANGELOG.md` at repo root in keepachangelog 1.1.0 format with seeded `[0.1.1] - 2026-04-11` entry (per D-16 / PKG-04)
- [ ] `CONTRIBUTING.md` at repo root with dev setup, PR conventions, issues, LICENSE link (per D-17 / PKG-04)
- [ ] `examples/hello.py` exists, imports `zeroth.core`, is env-gated on `ANTHROPIC_API_KEY` (per D-18 / D-19 / D-20 / PKG-06)
- [ ] `examples/hello.py` exits 0 with SKIP message when the key is absent (pitfall #6)
- [ ] `.planning/STATE.md` Regulus blocker removed; trusted-publisher entry reworded to zeroth-core-only covering both pypi.org and test.pypi.org (per D-25)
- [ ] `.planning/STATE.md last_updated` bumped to 2026-04-11
- [ ] No source code under `src/zeroth/` modified (this plan is metadata + fixture only)
</success_criteria>

<output>
After completion, create `.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-02-SUMMARY.md` covering: the three new metadata files (sizes, key sections), the hello.py approach taken (full graph vs litellm fallback and why), skip-path verification output, and the before/after of the STATE.md blockers/todos sections.
</output>
