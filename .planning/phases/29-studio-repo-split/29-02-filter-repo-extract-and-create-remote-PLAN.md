---
phase: 29-studio-repo-split
plan: 02
type: execute
wave: 2
depends_on: [29-01]
files_modified:
  - /tmp/zeroth-studio-split/**   # scratch directory OUTSIDE this repo
  - github:rrrozhd/zeroth-studio    # new public GitHub repo (created, not a file)
autonomous: true
requirements:
  - STUDIO-01
user_setup: []

must_haves:
  truths:
    - "A throwaway clone exists at /tmp/zeroth-studio-split built with git clone --no-local from /Users/dondoe/coding/zeroth"
    - "git filter-repo has rewritten the clone's history to contain only apps/studio, apps/studio-mockups, tests/studio"
    - "The filtered repo's working tree contains apps/studio/** and apps/studio-mockups/** (tests/studio is empty on disk — bytecode only — and may or may not materialize; history for the path IS preserved)"
    - "Public repo rrrozhd/zeroth-studio exists on GitHub with default branch main"
    - "The filtered history has been pushed to rrrozhd/zeroth-studio main"
  artifacts:
    - path: "/tmp/zeroth-studio-split/apps/studio/package.json"
      provides: "Proof that apps/studio survived the filter"
      contains: "zeroth-studio"
    - path: "/tmp/zeroth-studio-split/apps/studio/src/api/client.ts"
      provides: "Proof that the preflight Wave 1 changes (Plan 01, Task 2) carried through"
      contains: "import.meta.env.VITE_API_BASE_URL"
    - path: "/tmp/zeroth-studio-split/apps/studio/eslint.config.js"
      provides: "Proof that the preflight ESLint config carried through"
      min_lines: 15
    - path: "/tmp/zeroth-studio-split/apps/studio/nginx.conf"
      provides: "Proof that the preflight nginx.conf carried through"
      min_lines: 8
  key_links:
    - from: "/tmp/zeroth-studio-split"
      to: "github.com/rrrozhd/zeroth-studio"
      via: "git push -u origin main"
      pattern: "origin\\s+https?://.*rrrozhd/zeroth-studio"
---

<objective>
Wave 2: Extract the three paths (apps/studio, apps/studio-mockups, tests/studio) from the zeroth monorepo into a fresh scratch directory using git filter-repo, then create the public GitHub repo rrrozhd/zeroth-studio and push the filtered history as main.

**CRITICAL WORKING DIRECTORY NOTE:** This plan mutates `/tmp/zeroth-studio-split/` — a directory OUTSIDE the current working repo. The executor must `cd` into that scratch directory for filter-repo and git operations. Do NOT run filter-repo inside /Users/dondoe/coding/zeroth.

Purpose: Deliver STUDIO-01 (public repo with preserved git history) and set up the scratch directory that Plan 03 will finish wiring. This plan stops AFTER the first push — bootstrap files (LICENSE, CI, README) are added in Plan 03 so that the filter-repo step remains atomic and reviewable.

Output: A pushed rrrozhd/zeroth-studio repo on GitHub whose contents are byte-identical to /tmp/zeroth-studio-split/ at the moment of push.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/29-studio-repo-split/29-CONTEXT.md
@.planning/phases/29-studio-repo-split/29-RESEARCH.md
@.planning/phases/29-studio-repo-split/29-01-SUMMARY.md

<interfaces>
<!-- filter-repo command (verified in 29-RESEARCH §Pattern 1): -->
```bash
git filter-repo --path apps/studio --path apps/studio-mockups --path tests/studio
```

<!-- gh repo create signature (verified in 29-RESEARCH §Pitfall 2): -->
```bash
gh repo create rrrozhd/zeroth-studio --public \
  --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"
```
Do NOT pass --source or --push — filter-repo removes the origin remote, so we two-step manually.
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Fresh clone + git filter-repo extraction to /tmp/zeroth-studio-split</name>
  <files>/tmp/zeroth-studio-split/ (scratch directory outside this repo)</files>
  <action>
**Working directory for this task: /tmp (not the current repo)**

Preconditions: Plan 01 is committed in /Users/dondoe/coding/zeroth so its changes are in the history that filter-repo will read. Verify before starting:
```bash
cd /Users/dondoe/coding/zeroth
git log --oneline -- apps/studio/src/api/client.ts apps/studio/eslint.config.js apps/studio/nginx.conf openapi/zeroth-core-openapi.json scripts/dump_openapi.py | head
# Expect: recent commits from Plan 01 visible.
```

Then extract:
```bash
# 1. If a previous run left artifacts, nuke them
rm -rf /tmp/zeroth-studio-split

# 2. Fresh clone with --no-local (required — see 29-RESEARCH §Pitfall 1)
git clone --no-local /Users/dondoe/coding/zeroth /tmp/zeroth-studio-split
cd /tmp/zeroth-studio-split

# 3. Confirm filter-repo tool version
git filter-repo --version
# Expect: a40bce548d2c or similar (29-RESEARCH verified this install)

# 4. Run the extraction — multi-path, keeps history only for these three paths
git filter-repo \
  --path apps/studio \
  --path apps/studio-mockups \
  --path tests/studio

# 5. Sanity checks
git log --oneline | wc -l
# Expect: >= 32 (29-RESEARCH baseline); Plan 01 commits add more

ls -la
# Expect: apps/ present; tests/ may or may not be present (tests/studio has only bytecode);
# NO src/, NO .planning/, NO pyproject.toml, NO top-level README.md, etc.

ls apps/
# Expect: studio/ and studio-mockups/

# 6. Verify Wave 1 preflight survived
test -f apps/studio/eslint.config.js || { echo "FAIL: eslint.config.js missing"; exit 1; }
test -f apps/studio/nginx.conf || { echo "FAIL: nginx.conf missing"; exit 1; }
test -f apps/studio/.env.example || { echo "FAIL: .env.example missing"; exit 1; }
grep -q "import.meta.env.VITE_API_BASE_URL" apps/studio/src/api/client.ts || { echo "FAIL: client.ts not env-driven"; exit 1; }
grep -q '"typecheck"' apps/studio/package.json || { echo "FAIL: typecheck script missing"; exit 1; }

# 7. Broader audit — ensure nothing from zeroth-core crept through
# If filter-repo did its job, these should return nothing:
find . -name "pyproject.toml" -not -path "./.git/*" | grep -q . && { echo "FAIL: pyproject.toml leaked"; exit 1; } || true
find . -name "*.py" -not -path "./.git/*" | head
# Any .py files here should ONLY be historical content of tests/studio/, NOT
# zeroth-core source. If src/zeroth/** appears anywhere outside .git/, something is wrong.
find . -type d -name "zeroth" -not -path "./.git/*" | grep -q . && { echo "FAIL: src/zeroth directory leaked"; exit 1; } || true
```

If ANY sanity check fails, stop and diagnose before Task 2. filter-repo is not idempotent on a stale clone — you must `rm -rf /tmp/zeroth-studio-split` and retry from step 1.

Do NOT commit anything new in this task — the clone already has the full filtered history; Plan 03 handles bootstrap additions.
  </action>
  <verify>
    <automated>cd /tmp/zeroth-studio-split && test -f apps/studio/package.json && test -f apps/studio/eslint.config.js && test -f apps/studio/nginx.conf && test -f apps/studio/.env.example && grep -q "import.meta.env.VITE_API_BASE_URL" apps/studio/src/api/client.ts && test -d apps/studio-mockups && ! test -d src/zeroth && ! test -f pyproject.toml && [ "$(git log --oneline | wc -l | tr -d ' ')" -ge 32 ]</automated>
  </verify>
  <done>/tmp/zeroth-studio-split exists, history is rewritten, contains apps/studio + apps/studio-mockups, Wave 1 preflight files are present, no zeroth-core source leaked, git log has >=32 commits.</done>
</task>

<task type="auto">
  <name>Task 2: gh repo create rrrozhd/zeroth-studio + two-step push</name>
  <files>github:rrrozhd/zeroth-studio (remote — created, origin added, branch pushed)</files>
  <action>
**Working directory for this task: /tmp/zeroth-studio-split**

Per 29-RESEARCH §Pitfall 2, we two-step this: `gh repo create` does NOT take `--push` here because filter-repo removes the origin remote, so we add it manually.

```bash
# 0. Verify gh is authenticated (orchestrator pre-verified this, but double check)
gh auth status
# Expect: "Logged in to github.com account rrrozhd"

# 1. Create the public repo (no --source, no --push)
gh repo create rrrozhd/zeroth-studio \
  --public \
  --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"

# 2. Verify creation via API (idempotent check)
gh repo view rrrozhd/zeroth-studio --json name,isPrivate,defaultBranchRef \
  | python -c "import json,sys; d=json.load(sys.stdin); assert d['name']=='zeroth-studio' and d['isPrivate'] is False, d; print('OK:',d)"

# 3. Wire the local scratch repo to the new remote
cd /tmp/zeroth-studio-split
git remote -v
# Expect: nothing (filter-repo removed origin). If any origin is present, remove it:
git remote remove origin 2>/dev/null || true

git remote add origin https://github.com/rrrozhd/zeroth-studio.git
git branch -M main
git push -u origin main

# 4. Verify push
gh repo view rrrozhd/zeroth-studio --json defaultBranchRef \
  | python -c "import json,sys; d=json.load(sys.stdin); assert d['defaultBranchRef']['name']=='main', d; print('OK default branch:',d['defaultBranchRef']['name'])"

# 5. Verify remote commit count matches local
REMOTE_SHA=$(gh api repos/rrrozhd/zeroth-studio/commits/main --jq .sha)
LOCAL_SHA=$(git rev-parse HEAD)
[ "$REMOTE_SHA" = "$LOCAL_SHA" ] && echo "OK: remote==local $LOCAL_SHA" || { echo "FAIL: remote=$REMOTE_SHA local=$LOCAL_SHA"; exit 1; }
```

**Force-push warning:** This is the initial push to a fresh repo. Force-push is acceptable only if the remote is non-empty due to an unexpected template/autoinit; in practice `gh repo create` without `--clone` creates an empty repo and a plain push succeeds. If a plain push fails with "fetch first", inspect the remote — do NOT blindly force-push. Because the repo was just created, the only legitimate recovery is `git push -u origin main --force-with-lease`, and only after confirming the remote truly is fresh (empty other than README.md auto-commit if any).

Do not add LICENSE/README/CI yet — Plan 03 does that on a new branch that ultimately lands on main.
  </action>
  <verify>
    <automated>gh repo view rrrozhd/zeroth-studio --json name,isPrivate,defaultBranchRef | python -c "import json,sys; d=json.load(sys.stdin); assert d['name']=='zeroth-studio' and d['isPrivate'] is False and d['defaultBranchRef']['name']=='main'; print('OK')" && cd /tmp/zeroth-studio-split && [ "$(gh api repos/rrrozhd/zeroth-studio/commits/main --jq .sha)" = "$(git rev-parse HEAD)" ]</automated>
  </verify>
  <done>Public GitHub repo rrrozhd/zeroth-studio exists, default branch main matches local HEAD, no template/auto-init commits, ready for Plan 03 to add bootstrap files.</done>
</task>

</tasks>

<verification>
- /tmp/zeroth-studio-split contains filtered history, Wave 1 preflight files survived, no zeroth-core source leaked
- rrrozhd/zeroth-studio exists on GitHub as public, main branch matches local HEAD
- `gh repo view` confirms name, visibility, default branch
- Commit SHAs match between local scratch clone and remote main
- STUDIO-01 is satisfied by this plan alone (preserved history in a public repo)
</verification>

<success_criteria>
1. Filtered repo history length >= 32 commits (baseline per 29-RESEARCH)
2. No pyproject.toml, src/zeroth/, or tests/test_studio_api.py leaked into the scratch directory
3. Wave 1 preflight changes (ESLint config, env-driven client.ts, nginx.conf, split scripts, .env.example) are present at /tmp/zeroth-studio-split/apps/studio/
4. rrrozhd/zeroth-studio public repo exists with main branch pointing at the filtered HEAD
5. No destructive operations on /Users/dondoe/coding/zeroth — the source working tree is untouched
</success_criteria>

<output>
After completion, create `.planning/phases/29-studio-repo-split/29-02-SUMMARY.md` covering:
- Commit count in the filtered history (actual vs. 32 baseline)
- Scratch directory path and final HEAD SHA
- Remote URL and confirmed default branch
- Any anomalies during filter-repo (e.g. unexpected files that required investigation)
- Confirmation that /Users/dondoe/coding/zeroth was not mutated
</output>
