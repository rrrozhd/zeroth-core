---
phase: 11-config-postgres-storage
plan: 02
subsystem: storage, repositories, service
tags: [async, database, repositories, orchestrator, dispatch]
dependency_graph:
  requires: [AsyncDatabase, AsyncConnection, create_database]
  provides: [async-repositories, async-bootstrap, async-orchestrator, async-dispatch]
  affects: [graph, runs, contracts, deployments, approvals, audit, dispatch, service]
tech_stack:
  added: []
  patterns: [async-repository-pattern, async-service-layer]
key_files:
  created: []
  modified:
    - src/zeroth/graph/repository.py
    - src/zeroth/runs/repository.py
    - src/zeroth/contracts/registry.py
    - src/zeroth/deployments/repository.py
    - src/zeroth/approvals/repository.py
    - src/zeroth/audit/repository.py
    - src/zeroth/agent_runtime/thread_store.py
    - src/zeroth/dispatch/lease.py
    - src/zeroth/guardrails/rate_limit.py
    - src/zeroth/service/bootstrap.py
    - src/zeroth/service/app.py
    - src/zeroth/orchestrator/runtime.py
    - src/zeroth/dispatch/worker.py
    - src/zeroth/approvals/service.py
    - src/zeroth/deployments/service.py
    - src/zeroth/guardrails/dead_letter.py
    - src/zeroth/service/authorization.py
    - src/zeroth/service/auth.py
    - src/zeroth/service/run_api.py
    - src/zeroth/service/admin_api.py
    - src/zeroth/service/approval_api.py
    - src/zeroth/service/contracts_api.py
    - src/zeroth/service/audit_api.py
    - src/zeroth/observability/queue_gauge.py
    - src/zeroth/__init__.py
decisions:
  - All repository constructors remain synchronous (just store AsyncDatabase ref); only data methods are async
  - Alembic migrations run via sync run_migrations() before async database creation (Alembic uses SQLAlchemy sync engine internally)
  - DeploymentService, DeadLetterManager, and all service API helpers converted to async alongside repositories
  - Authorization helpers (require_permission, require_deployment_scope, require_resource_scope) made async to support async audit writes on denial
metrics:
  duration: 1523s
  completed: 2026-04-06
  tasks_completed: 2
  tasks_total: 2
  files_modified: 25
requirements:
  - CFG-02
---

# Phase 11 Plan 02: Async Repository Conversion Summary

Complete async rewrite of all 7 repositories, LeaseManager, guardrail classes, and all callers (bootstrap, orchestrator, dispatch worker, approval service, deployment service, all HTTP API routes) from synchronous SQLiteDatabase to async AsyncDatabase protocol.

## What Was Done

### Task 1: Rewrite all 7 repositories and infrastructure classes to async
- Converted GraphRepository, RunRepository, ThreadRepository, ContractRegistry, SQLiteDeploymentRepository, ApprovalRepository, AuditRepository to use AsyncDatabase
- Converted LeaseManager, TokenBucketRateLimiter, QuotaEnforcer to use AsyncDatabase
- Converted RepositoryThreadResolver and RepositoryThreadStateStore to async
- Removed all MIGRATIONS lists, SCHEMA_SCOPE constants, and apply_migrations() calls from repositories (Alembic handles migrations at startup)
- Removed sqlite3 import from audit repository (replaced IntegrityError check with fetch_one existence check)
- All SQL continues using ? placeholders for dialect-agnostic queries
- **Commit:** 884fb0a

### Task 2: Update all callers to async
- Made bootstrap_service() async, taking AsyncDatabase parameter; added run_migrations() for Alembic startup
- Converted RuntimeOrchestrator: run_graph, resume_graph, _drive, _record_history, _record_failed_execution_audit, _enforce_policy, _enforce_loop_guards, _fail_run, _resolve_thread, _gate_policy_required_side_effects, _consume_side_effect_approval, record_approval_resolution all now async with await on repo/audit/approval calls
- Converted RunWorker: start, poll_loop, _drive_run, _drive_approval_resumed, _mark_failed, _renewal_loop all now await async lease_manager and run_repository calls
- Converted ApprovalService: all methods async with await on repository calls
- Converted DeploymentService: all methods async with await on graph_repository and deployment_repository calls
- Converted DeadLetterManager: handle_run_failure now async
- Updated all HTTP API route handlers (run_api, admin_api, approval_api, contracts_api, audit_api) to await repository and service method calls
- Converted authorization helpers (require_permission, require_deployment_scope, require_resource_scope) to async
- Converted record_service_denial to async
- Updated QueueDepthGauge to await count_pending
- Updated zeroth __init__.py exports with new async types
- **Commit:** b853444

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] DeploymentService required async conversion**
- **Found during:** Task 2
- **Issue:** DeploymentService methods call async repository methods but were synchronous
- **Fix:** Converted all DeploymentService methods to async
- **Files modified:** src/zeroth/deployments/service.py

**2. [Rule 3 - Blocking] DeadLetterManager required async conversion**
- **Found during:** Task 2
- **Issue:** DeadLetterManager.handle_run_failure calls async run_repository methods
- **Fix:** Converted handle_run_failure to async, removed run_in_executor wrapper in worker
- **Files modified:** src/zeroth/guardrails/dead_letter.py, src/zeroth/dispatch/worker.py

**3. [Rule 3 - Blocking] All HTTP API route handlers and helpers required async updates**
- **Found during:** Task 2
- **Issue:** Service API routes called now-async repository/service methods synchronously
- **Fix:** Added await to all repository and service calls in route handlers, converted helper functions (authorization, audit visibility, deployment context) to async
- **Files modified:** All service API files (run_api, admin_api, approval_api, contracts_api, audit_api, authorization, auth)

## Known Stubs

None -- all implementations are functional.

## Verification Results

1. `uv run ruff check src/` -- all checks passed, zero lint errors
2. All repository files confirmed to use `async def` for data methods and `database: AsyncDatabase` constructors
3. No `SQLiteDatabase` imports remain outside storage module itself
4. All callers confirmed to use `await` for async method calls

## Self-Check: PASSED
