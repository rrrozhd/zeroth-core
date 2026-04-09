# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.1 — Production Readiness

**Shipped:** 2026-04-09
**Phases:** 11 | **Plans:** 30

### What Was Built
- Real LLM provider integration via LiteLLM with exponential backoff retry and token usage capture
- Regulus economics: per-call cost events, per-tenant budget enforcement, cost REST endpoints
- 8 memory connector types (3 in-memory, 2 Redis, 3 vector/search) bridged to GovernAI v0.3.0 protocol
- Docker sandbox sidecar with per-execution network isolation
- Durable webhooks with HMAC signing, dead-letter store, and approval SLA escalation
- Production deployment: multi-stage Dockerfile, 6-service docker-compose, Nginx TLS, health probes
- Native LLM API parity: tool schemas, structured output, model params, MCP server connections
- Postgres storage backend with Alembic migrations, ARQ wakeup, horizontal worker scaling

### What Worked
- Phase-based parallel development with worktree isolation allowed independent progress on 11 phases
- Gap closure phases (18, 20, 21) effectively caught integration issues post-merge
- Milestone audit workflow identified real integration gaps (INT-01, INT-02, INT-03) that would have been missed
- GovernAI protocol alignment kept memory connectors interchangeable across all backends

### What Was Inefficient
- Some ROADMAP.md plan checkboxes drifted from actual completion status (11-02, 12-01, 13-02 unchecked despite work complete)
- SUMMARY.md frontmatter `requirements_completed` was inconsistently populated across ~20 requirements
- Phase 18 marked "In Progress" in ROADMAP despite verification passing — documentation staleness accumulated
- 3 gap closure phases (18, 20, 21) needed after initial 8 phases — integration wiring should be considered earlier

### Patterns Established
- Milestone audit (requirement/integration/flow cross-reference) as gate before completion
- Gap closure phases numbered sequentially after initial phases rather than inserted
- Tech debt tracked as non-blocking audit items separate from requirement gaps
- Bootstrap wiring as dedicated phase pattern for connecting independently developed subsystems

### Key Lessons
1. Worktree-isolated parallel development creates merge/wiring gaps — plan an integration wiring phase from the start
2. Documentation metadata (checkboxes, frontmatter) needs automated verification, not manual tracking
3. Audit-driven gap closure is effective but reactive — proactive integration testing between phases would be cheaper
4. Fail-open patterns (budget enforcement when Regulus is down) are the right default for optional companion services

### Cost Observations
- 168 commits across 4 days
- 350 files changed, +47,444 / -3,273 lines
- Codebase: 21,928 LOC source + 18,035 LOC tests

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Phases | Plans | Key Change |
|-----------|--------|-------|------------|
| v1.0 | 9 | 13 | Foundation: graph, orchestration, governance, deployment |
| v1.1 | 11 | 30 | Production hardening: real providers, economics, infra, deployment |

### Top Lessons (Verified Across Milestones)

1. Phase-based planning with explicit success criteria prevents scope creep
2. Integration wiring between independently developed subsystems needs its own phase
