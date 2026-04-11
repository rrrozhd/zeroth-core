# Phase 28: PyPI Publishing - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in 28-CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-11
**Phase:** 28 — PyPI Publishing (`econ-instrumentation-sdk` + `zeroth-core`)
**Areas discussed:** Optional extras split, Version + release strategy, Trusted-publisher CI design, PKG-06 acceptance + econ-sdk reconciliation, Packaging hardening

---

## Gray Area Selection

| Option | Description | Selected |
|--------|-------------|----------|
| Optional extras split | Carve [memory-pg]/[memory-chroma]/[memory-es]/[dispatch]/[sandbox]/[all] out of the monolithic dependencies list | ✓ |
| Version + release strategy | First version, SemVer/CalVer, tag vs dispatch trigger, TestPyPI dry-run | ✓ |
| Trusted-publisher CI design | Combined vs separate workflows, GH environment, pre-publish gates | ✓ |
| PKG-06 acceptance + econ-sdk reconciliation | Hello example fixture + STATE/pyproject mismatch on econ-sdk | ✓ |

---

## Optional Extras Split

### Q1: What stays in the base `dependencies`?

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal core | Only universal deps; all backend-specific packages move to extras | ✓ |
| Minimal + Postgres default | Same as minimal but psycopg+pgvector stay in core | |
| Current (no split) | Leave everything as-is, extras as empty aliases | |

### Q2: What does `[all]` install?

| Option | Description | Selected |
|--------|-------------|----------|
| Everything except memory (mislabeled — actually all backends incl. memory) | dispatch + sandbox + memory-pg + memory-chroma + memory-es | ✓ |
| Everything with Postgres memory only | dispatch + sandbox + memory-pg | |
| Literally all extras | Mechanical union of every extra | |

**Note:** Label was confusing. Captured per description: `[all]` includes all three memory backends side-by-side (no import conflict; runtime config picks one).

### Q3: Extra names refinement?

| Option | Description | Selected |
|--------|-------------|----------|
| Locked as specified | Use PKG-03 names verbatim | ✓ |
| Refine to consistent scheme | Drop `memory-` prefix | |

### Q4: Where do redis/arq belong?

| Option | Description | Selected |
|--------|-------------|----------|
| Both in [dispatch] | One extra enables distributed-worker path | ✓ |
| redis in core, arq in [dispatch] | Slightly larger base install | |
| Both in core | Fails PKG-03 intent | |

---

## Version + Release Strategy

### Q1: First PyPI version?

| Option | Description | Selected |
|--------|-------------|----------|
| 0.1.0 stable | Matches current pyproject; pre-1.0 signal | ✓ |
| 0.1.0a1 (alpha) | Soft launch hidden from default pip | |
| 0.2.0 | Bump to mark post-rename milestone | |

### Q2: Release trigger?

| Option | Description | Selected |
|--------|-------------|----------|
| GitHub Release → tag | release-published event triggers workflow | ✓ |
| Tag push only | Lightweight, no Release UI | |
| Manual workflow_dispatch | Operator-driven from Actions tab | |

### Q3: TestPyPI dry-run?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — staging job in same workflow | publish-testpypi → smoke → publish-pypi | ✓ |
| No — straight to PyPI | Single publish step | |
| Manual: separate workflow for TestPyPI | Operator-driven staging | |

### Q4: Version source of truth?

| Option | Description | Selected |
|--------|-------------|----------|
| pyproject.toml [project].version (static) | Hand-edited; tag asserted to match | ✓ |
| hatch-vcs from git tag | Auto-derived; no hand-edit | |
| src/zeroth/core/__version__.py | Importable runtime version | |

---

## Trusted-Publisher CI Design

### Q1: Combined or separate workflows?

| Option | Description | Selected |
|--------|-------------|----------|
| Separate workflows | release-zeroth-core.yml only; econ-sdk owned by Regulus | ✓ |
| One combined workflow | Doesn't fit — econ-sdk source not in this repo | |

### Q2: GitHub environment?

| Option | Description | Selected |
|--------|-------------|----------|
| `pypi` env, no approval | Standard pattern; release gated by who can tag | ✓ |
| `pypi` + `testpypi`, both require approval | Belt-and-braces, adds friction | |
| `release` env, single name | Less granular OIDC scoping | |

### Q3: Pre-publish gates?

| Option | Description | Selected |
|--------|-------------|----------|
| build + smoke-install + tests | Maximum confidence, catches packaging bugs | ✓ |
| build + tests only | Skip wheel-install smoke | |
| build only | Trust main is green | |

### Q4: Sigstore + SBOM?

| Option | Description | Selected |
|--------|-------------|----------|
| Sigstore attestation only | `attestations: true` flag, no extra infra | ✓ |
| Attestation + SBOM (cyclonedx) | Heavier setup | |
| Neither for v0.1.0 | Skip both | |

---

## PKG-06 Acceptance + econ-sdk Reconciliation

### Q1: How to satisfy PKG-06 without waiting for Phase 30 docs?

| Option | Description | Selected |
|--------|-------------|----------|
| Ship `examples/hello.py` in repo | Real script; CI runs it from clean venv | ✓ |
| Block on Phase 30 | Violates Phase 28 success criterion | |
| Inline acceptance test only | No user-facing example file | |

### Q2: Is PKG-01 (econ-sdk on PyPI) done?

| Option | Description | Selected |
|--------|-------------|----------|
| Done — STATE.md is stale | pyproject pins >=0.1.1 from PyPI; reconcile state | ✓ |
| Partly done — needs version bump | Phase 28 includes Regulus workstream | |
| Not done — placeholder version | Phase 28 must coordinate Regulus work | |

### Q3: License?

| Option | Description | Selected |
|--------|-------------|----------|
| Apache-2.0 | Permissive + patent grant + contributor protection | ✓ |
| MIT | Shortest, no patent grant | |
| BSL-1.1 → Apache-2.0 | Source-available, commercial-use restricted | |

### Q4: CHANGELOG/CONTRIBUTING depth?

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal but real | One CHANGELOG entry; 1-page CONTRIBUTING | ✓ |
| Full OSS treatment | Heavyweight — belongs in Phase 30/31 | |
| Stubs only | Satisfies letter, not spirit | |

---

## Packaging Hardening (clarifying)

### Q1: What does `examples/hello.py` exercise?

| Option | Description | Selected |
|--------|-------------|----------|
| Single agent + echo LLM | No API key needed; perfect for CI | |
| Single agent + real LLM (env-gated) | Closer to real usage; CI skips without key | ✓ |
| Tool-using agent + approval gate | Bigger; belongs in Phase 30 | |

### Q2: Fix wheel target `packages = ["src/zeroth"]` → `["src/zeroth/core"]`?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, fix as part of packaging hardening | Verify wheel contents post-fix | ✓ |
| Investigate first — may already work | Add a research task | |
| Out of scope — backport to Phase 27 | Risky | |

---

*End of discussion log.*
