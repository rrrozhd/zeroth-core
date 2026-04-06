# Architecture

**Analysis Date:** 2026-04-05

## Pattern Overview

**Overall:** Modular Monolith with Domain-Driven Boundaries

**Key Characteristics:**
- Single deployable Python package (`src/zeroth/`) composed of ~22 domain modules with explicit public APIs via `__init__.py` barrel exports
- Dependency on local `governai` library for base runtime primitives (`RunState`, `RunStatus`, `GovernedFlowSpec`, etc.)
- Deployment-scoped service instances: each FastAPI app serves exactly one deployment (one published graph snapshot)
- SQLite as the primary persistence layer (WAL mode, migration-managed schemas), with Redis support for distributed runtime stores
- Synchronous repository pattern for all data access; async only at the orchestrator execution loop and HTTP layers
- Explicit object construction for dependency injection -- no DI framework, just a `ServiceBootstrap` dataclass wired by a factory function

## Layers

**HTTP / API Layer:**
- Purpose: Thin REST API surface that accepts requests and delegates to domain services
- Location: `src/zeroth/service/`
- Contains: FastAPI app factory, route registrations, request/response models, authentication middleware
- Key files:
  - `src/zeroth/service/app.py` -- FastAPI app factory (`create_app()`), auth middleware, health endpoint, lifespan (worker start/stop)
  - `src/zeroth/service/run_api.py` -- `POST /runs` (create run), `GET /runs/{id}` (status)
  - `src/zeroth/service/approval_api.py` -- Approval listing and resolution endpoints
  - `src/zeroth/service/audit_api.py` -- Audit timeline and evidence endpoints
  - `src/zeroth/service/contracts_api.py` -- Contract listing
  - `src/zeroth/service/admin_api.py` -- Admin and metrics endpoints
  - `src/zeroth/service/auth.py` -- `ServiceAuthenticator`: API key + JWT bearer authentication
  - `src/zeroth/service/authorization.py` -- RBAC permission checks (`require_permission()`, `require_deployment_scope()`)
- Depends on: All domain modules (via bootstrap wiring)
- Used by: External HTTP clients

**Bootstrap / Composition Root:**
- Purpose: Dependency injection container that wires all domain modules together for a specific deployment
- Location: `src/zeroth/service/bootstrap.py`
- Contains: `ServiceBootstrap` dataclass (holds all wired dependencies), `bootstrap_service()` factory, `bootstrap_app()` convenience function
- Pattern: `bootstrap_service()` takes a `SQLiteDatabase` + `deployment_ref`, constructs all repositories and services, returns the `ServiceBootstrap` container. The HTTP layer reads dependencies from `app.state.bootstrap`.
- Depends on: Every domain module
- Used by: `src/zeroth/service/app.py`, test harnesses

**Orchestration Layer:**
- Purpose: Core execution engine that drives graph workflows step-by-step
- Location: `src/zeroth/orchestrator/runtime.py`
- Contains: `RuntimeOrchestrator` -- the main loop that walks through graph nodes
- Key methods:
  - `run_graph()` -- starts a fresh execution from initial input
  - `resume_graph()` -- resumes a paused/interrupted run
  - `_drive()` -- main loop: pop pending node -> enforce guards -> check policy -> dispatch -> record audit -> plan next -> queue next -> checkpoint
  - `_dispatch_node()` -- routes to `AgentRunner` or `ExecutableUnitRunner` based on node type
  - `record_approval_resolution()` -- processes human approval decisions and resumes flow
- Depends on: `agent_runtime`, `execution_units`, `conditions`, `mappings`, `policy`, `approvals`, `audit`, `runs`, `secrets`, `graph`
- Used by: `service` layer (via bootstrap), `dispatch` layer (RunWorker)

**Durable Dispatch Layer:**
- Purpose: Reliable background execution with lease-based concurrency control
- Location: `src/zeroth/dispatch/`
- Key files:
  - `src/zeroth/dispatch/worker.py` -- `RunWorker`: polls for PENDING runs, claims via lease, drives through orchestrator, handles orphan recovery
  - `src/zeroth/dispatch/lease.py` -- `LeaseManager`: SQLite-backed atomic lease acquisition (optimistic locking via UPDATE), renewal, release, orphan claiming
- Pattern: Worker runs as a single asyncio background task. Uses a `Semaphore` for bounded concurrency (`max_concurrency=8`). Lease renewal runs as a parallel asyncio task for each active run. On startup, claims orphaned RUNNING runs with expired leases.
- Depends on: `runs`, `orchestrator`, `graph`, `storage`, `guardrails`
- Used by: Started in app lifespan (`app.py`)

**Graph Domain:**
- Purpose: Defines the workflow graph data model and persistence
- Location: `src/zeroth/graph/`
- Key files:
  - `src/zeroth/graph/models.py` -- `Graph`, `Node` (discriminated union of `AgentNode | ExecutableUnitNode | HumanApprovalNode`), `Edge`, `Condition`, `ExecutionSettings`, `DisplayMetadata`
  - `src/zeroth/graph/repository.py` -- `GraphRepository`: save/get/list/publish/archive/clone/diff
  - `src/zeroth/graph/serialization.py` -- JSON serialize/deserialize for graph storage
  - `src/zeroth/graph/versioning.py` -- `clone_graph_version()`, `graph_version_ref()`
  - `src/zeroth/graph/validation.py` -- Structural validation rules
  - `src/zeroth/graph/validation_errors.py` -- Validation error types
  - `src/zeroth/graph/diff.py` -- Version comparison
  - `src/zeroth/graph/storage.py` -- Schema migrations for graph tables
- Graph lifecycle: `DRAFT` -> `PUBLISHED` -> `ARCHIVED` (one-way transitions)
- Depends on: `storage`, `mappings` (for `EdgeMapping` model), `governai` (for `GovernedFlowSpec`)
- Used by: `orchestrator`, `deployments`, `service`

**Agent Runtime:**
- Purpose: Everything needed to execute AI agent nodes
- Location: `src/zeroth/agent_runtime/`
- Key files:
  - `src/zeroth/agent_runtime/runner.py` -- `AgentRunner`: callable that executes a single agent step
  - `src/zeroth/agent_runtime/provider.py` -- `ProviderAdapter` protocol + `GovernedLLMProviderAdapter`, `DeterministicProviderAdapter`
  - `src/zeroth/agent_runtime/tools.py` -- Tool attachment system: `ToolAttachmentManifest`, `ToolAttachmentRegistry`, `ToolAttachmentBridge`, permission enforcement (`ToolPermissionError`, `UndeclaredToolError`)
  - `src/zeroth/agent_runtime/prompt.py` -- `PromptAssembler`: builds prompts from config, instructions, memory context
  - `src/zeroth/agent_runtime/thread_store.py` -- `RepositoryThreadResolver`, `RepositoryThreadStateStore` for conversation persistence
  - `src/zeroth/agent_runtime/validation.py` -- `OutputValidator`: validates agent outputs against contracts
  - `src/zeroth/agent_runtime/models.py` -- `AgentConfig`, `AgentRunResult`, `PromptAssembly`, `RetryPolicy`
  - `src/zeroth/agent_runtime/errors.py` -- Error hierarchy: `AgentRuntimeError` -> timeout, provider, validation, retry errors
- Depends on: `contracts`, `memory`, `governai`
- Used by: `orchestrator` (via `agent_runners` mapping keyed by node_id)

**Execution Units:**
- Purpose: Executes deterministic code/script steps (non-AI nodes)
- Location: `src/zeroth/execution_units/`
- Key files:
  - `src/zeroth/execution_units/runner.py` -- `ExecutableUnitRunner`, `ExecutableUnitRegistry`, `ExecutableUnitBinding`
  - `src/zeroth/execution_units/sandbox.py` -- `SandboxManager`, `DockerSandboxConfig`, strictness modes, environment caching
  - `src/zeroth/execution_units/models.py` -- `ExecutableUnitManifest` variants (Native, WrappedCommand, Project)
  - `src/zeroth/execution_units/adapters.py` -- `PythonRuntimeAdapter`, `CommandRuntimeAdapter`
  - `src/zeroth/execution_units/integrity.py` -- `AdmissionController`: digest-based manifest verification
  - `src/zeroth/execution_units/io.py` -- Input injection, output extraction (JSON stdout, file-based)
  - `src/zeroth/execution_units/constraints.py` -- `ResourceConstraints`, Docker resource flags
  - `src/zeroth/execution_units/validator.py` -- `ExecutableUnitValidator`
- Depends on: `secrets`, `contracts`
- Used by: `orchestrator`

**Conditions / Branching:**
- Purpose: Evaluates edge conditions to decide which path to take at runtime
- Location: `src/zeroth/conditions/`
- Key files:
  - `src/zeroth/conditions/branch.py` -- `NextStepPlanner`, `BranchResolver`, `BranchResolution`
  - `src/zeroth/conditions/evaluator.py` -- `ConditionEvaluator`
  - `src/zeroth/conditions/models.py` -- `ConditionContext`, `TraversalState`, `ConditionOutcome`
  - `src/zeroth/conditions/binding.py` -- `ConditionBinder`
  - `src/zeroth/conditions/recorder.py` -- `ConditionResultRecorder`
- Depends on: `graph` (for Condition/Edge models)
- Used by: `orchestrator`

**Policy Layer:**
- Purpose: Capability-based access control for nodes at runtime
- Location: `src/zeroth/policy/`
- Key files:
  - `src/zeroth/policy/guard.py` -- `PolicyGuard`: evaluates allowed/denied capabilities, secret access, network mode, sandbox strictness, side-effect approval requirements
  - `src/zeroth/policy/models.py` -- `PolicyDefinition`, `Capability` enum, `EnforcementResult`, `PolicyDecision`
  - `src/zeroth/policy/registry.py` -- `PolicyRegistry`, `CapabilityRegistry`
- Pattern: Policies bound to graphs/nodes via string refs. Guard resolves refs through registries. Multiple policies combine via strictest-wins (intersection of allow-lists).
- Depends on: `graph`, `runs`
- Used by: `orchestrator`

**Approvals:**
- Purpose: Human-in-the-loop approval gates within graph execution
- Location: `src/zeroth/approvals/`
- Key files:
  - `src/zeroth/approvals/service.py` -- `ApprovalService`: create pending, resolve, get
  - `src/zeroth/approvals/repository.py` -- `ApprovalRepository` (SQLite-backed)
  - `src/zeroth/approvals/models.py` -- `ApprovalRecord`, `ApprovalResolution`, `ApprovalDecision`, `ApprovalStatus`
- Depends on: `runs`, `audit`, `storage`
- Used by: `orchestrator`, `service` (approval API)

**Audit:**
- Purpose: Append-only audit trail for every node execution, policy decision, approval action, and auth denial
- Location: `src/zeroth/audit/`
- Key files:
  - `src/zeroth/audit/models.py` -- `NodeAuditRecord`, `AuditTimeline`, `AuditQuery`, `AuditContinuityReport`, `ToolCallRecord`, `MemoryAccessRecord`
  - `src/zeroth/audit/repository.py` -- `AuditRepository` (SQLite-backed, append-only)
  - `src/zeroth/audit/evidence.py` -- Evidence bundle assembly for compliance
  - `src/zeroth/audit/verifier.py` -- Digest-chain integrity verification
  - `src/zeroth/audit/timeline.py` -- Timeline construction from audit records
  - `src/zeroth/audit/sanitizer.py` -- Sensitive data redaction
- Depends on: `identity`, `storage`
- Used by: `orchestrator`, `service`

**Runs / Threads:**
- Purpose: Execution state persistence
- Location: `src/zeroth/runs/`
- Key files:
  - `src/zeroth/runs/models.py` -- `Run` (extends GovernAI `RunState`), `Thread`, `RunHistoryEntry`, `RunConditionResult`, `RunFailureState`
  - `src/zeroth/runs/repository.py` -- `RunRepository` (CRUD, transitions, checkpoints, count_pending), `ThreadRepository`
- `Run` fields: `run_id`, `status`, `deployment_ref`, `graph_version_ref`, `tenant_id`, `current_node_ids`, `pending_node_ids`, `execution_history`, `node_visit_counts`, `condition_results`, `audit_refs`, `final_output`, `failure_state`, lease columns
- Depends on: `identity`, `storage`, `governai`
- Used by: `orchestrator`, `dispatch`, `service`

**Deployments:**
- Purpose: Immutable snapshots of published graphs for production execution
- Location: `src/zeroth/deployments/`
- Key files:
  - `src/zeroth/deployments/models.py` -- `Deployment`: contains `serialized_graph`, pinned contract versions, content digests (`graph_snapshot_digest`, `contract_snapshot_digest`, `attestation_digest`)
  - `src/zeroth/deployments/service.py` -- `DeploymentService`: creates deployment from published graph
  - `src/zeroth/deployments/repository.py` -- `SQLiteDeploymentRepository`
  - `src/zeroth/deployments/provenance.py` -- Provenance tracking
- Depends on: `graph`, `contracts`, `storage`
- Used by: `service` (bootstrap)

**Contracts:**
- Purpose: Versioned data schema registry for input/output validation
- Location: `src/zeroth/contracts/`
- Key files:
  - `src/zeroth/contracts/registry.py` -- `ContractRegistry` (SQLite-backed), `ContractVersion`, `ContractReference`, `StepContractBinding`, `ToolContractBinding`
  - `src/zeroth/contracts/errors.py` -- `ContractNotFoundError`, `ContractRegistryError`
- Depends on: `storage`
- Used by: `service`, `deployments`, `agent_runtime`, `execution_units`

**Identity:**
- Purpose: Shared authentication and authorization primitives
- Location: `src/zeroth/identity/models.py`
- Contains: `ActorIdentity`, `AuthenticatedPrincipal` (extends `ActorIdentity` with `credential_id`, `claims`), `AuthMethod` (API_KEY, BEARER), `ServiceRole` (OPERATOR, REVIEWER, ADMIN), `PrincipalScope`
- Depends on: None (leaf module)
- Used by: `service`, `audit`, `runs`, `approvals`

**Secrets:**
- Purpose: Secret resolution and redaction for audit safety
- Location: `src/zeroth/secrets/`
- Key files:
  - `src/zeroth/secrets/provider.py` -- `SecretProvider` protocol, `EnvSecretProvider`, `SecretResolver`
  - `src/zeroth/secrets/redaction.py` -- `SecretRedactor`: replaces known secret values in audit payloads
- Depends on: `execution_units` (for `EnvironmentVariable` model)
- Used by: `orchestrator`, `execution_units`

**Memory:**
- Purpose: Pluggable memory connectors for agent state persistence
- Location: `src/zeroth/memory/`
- Contains: `MemoryConnector` interface, `KeyValueMemoryConnector`, `ThreadMemoryConnector`, `RunEphemeralMemoryConnector`, `InMemoryConnectorRegistry`, `MemoryConnectorResolver`
- Depends on: None (leaf module)
- Used by: `agent_runtime`

**Mappings:**
- Purpose: Data transformation between graph nodes via edge mappings
- Location: `src/zeroth/mappings/`
- Contains: `EdgeMapping` model, `MappingExecutor`, `MappingValidator`, operation types (`PassthroughMappingOperation`, `RenameMappingOperation`, `ConstantMappingOperation`, `DefaultMappingOperation`)
- Depends on: None (leaf module)
- Used by: `orchestrator`, `graph`

**Guardrails:**
- Purpose: Operational safety limits
- Location: `src/zeroth/guardrails/`
- Contains: `GuardrailConfig` (backpressure depth, rate limit capacity/refill, daily quota, max concurrency, max failure count), `TokenBucketRateLimiter` (SQLite-backed), `QuotaEnforcer` (SQLite-backed), `DeadLetterManager`
- Depends on: `runs`, `storage`
- Used by: `service` (pre-accept guardrail checks in `run_api.py`), `dispatch` (dead-letter on repeated failures)

**Observability:**
- Purpose: Correlation IDs, metrics collection, queue depth monitoring
- Location: `src/zeroth/observability/`
- Contains: `MetricsCollector` (in-memory counters/histograms), correlation ID context vars (`get_correlation_id`, `set_correlation_id`), `QueueDepthGauge` (periodic PENDING count sampling)
- Depends on: `runs`
- Used by: `service`, `dispatch`

**Storage:**
- Purpose: Shared database and caching backends
- Location: `src/zeroth/storage/`
- Key files:
  - `src/zeroth/storage/sqlite.py` -- `SQLiteDatabase`: connection management (WAL, FK, NORMAL sync), `Migration` model, `apply_migrations()`, `transaction()` context manager, `EncryptedField` (Fernet-based)
  - `src/zeroth/storage/redis.py` -- `RedisConfig`, `GovernAIRedisRuntimeStores`, deployment modes (local/Docker/remote)
  - `src/zeroth/storage/json.py` -- JSON serialization helpers
- Depends on: `governai` (for Redis store types), `cryptography` (for Fernet)
- Used by: Every repository module

**Studio (emerging):**
- Purpose: Workflow authoring control plane (in-progress, Phase 10)
- Location: `src/zeroth/studio/` (sub-packages: `leases/`, `workflows/`)
- Frontend mockup: `apps/studio-mockups/` (Vue 3 + Vite)
- Status: Early development, no `__init__.py` at package root yet

## Data Flow

**Run Invocation (HTTP -> Execution -> Completion):**

1. Client sends `POST /runs` with `input_payload` and optional `thread_id`
2. Auth middleware in `app.py` authenticates via `ServiceAuthenticator.authenticate_headers()` (API key or JWT bearer)
3. `run_api.py` checks RBAC permission (`RUN_CREATE`), validates deployment scope (tenant/workspace match)
4. Input payload validated against deployment-pinned contract version via `ContractRegistry.resolve_model_type()`
5. Guardrail checks: backpressure (pending queue depth), rate limit (token bucket), daily quota
6. `Run` created in SQLite with status `PENDING`, entry node in `pending_node_ids`
7. `RunWorker.poll_loop()` claims the run via `LeaseManager.claim_pending()` (atomic SQL UPDATE with lease columns)
8. Worker transitions run to `RUNNING`, starts lease renewal background task, calls `RuntimeOrchestrator._drive()`
9. Orchestrator main loop per node:
   - Pop next pending node from queue
   - Enforce loop guards (max steps, max runtime)
   - Check/consume pending approval state
   - Evaluate policy guard (capabilities, secrets, network mode)
   - Gate side-effect approval if policy requires it
   - Dispatch: `AgentRunner.run()` for agent nodes, `ExecutableUnitRunner.run()` for executable units
   - Record history entry + audit record (with secret redaction)
   - Increment node visit count
   - Plan next nodes via `NextStepPlanner.plan()` (condition evaluation, branch resolution)
   - Queue next nodes with edge mapping transformations
   - Persist run state + checkpoint
10. On completion (no more pending nodes): status = `COMPLETED`, `final_output` set
11. Worker releases lease via `LeaseManager.release_lease()`
12. Client polls `GET /runs/{id}` to retrieve status and output

**Approval Gate Flow:**

1. Orchestrator encounters `HumanApprovalNode` or policy-required side-effect gate
2. `ApprovalService.create_pending()` stores an `ApprovalRecord`
3. Run status set to `WAITING_APPROVAL`, approval metadata stored in run, node re-inserted at front of pending queue
4. Run checkpointed; worker releases control
5. External reviewer retrieves approval via API, resolves with decision + optional payload edits
6. Run is re-queued (via `schedule_continuation`), worker picks it up
7. `record_approval_resolution()` records outcome, plans next nodes, resumes orchestration

**Deployment Creation:**

1. Graph authored and saved as `DRAFT` via `GraphRepository.save()`
2. Published via `GraphRepository.publish()` (transitions to `PUBLISHED`, immutable thereafter)
3. `DeploymentService` creates `Deployment` snapshot: serialized graph, pinned contract versions, content digests
4. `bootstrap_service()` loads deployment, deserializes graph, verifies graph_id/version/ref match, wires all services

**Crash Recovery:**

1. Worker starts, calls `LeaseManager.claim_orphaned()` -- finds RUNNING runs with expired leases
2. Each orphaned run is assigned to this worker with a fresh lease
3. Latest checkpoint is loaded via `recovery_checkpoint_id`
4. `orchestrator.resume_graph()` continues from the last persisted state

**State Management:**
- Run state persisted to SQLite after every node via `RunRepository.put()` + `write_checkpoint()`
- No in-memory-only state; full crash recovery through checkpoints
- Thread state groups related runs across conversation turns
- All state mutations go through repository methods

## Key Abstractions

**Graph (workflow definition):**
- Purpose: Declarative multi-step agent workflow
- Location: `src/zeroth/graph/models.py`
- Pattern: Pydantic model with `nodes` (discriminated union via `node_type`), `edges` (with optional `Condition` and `EdgeMapping`), `ExecutionSettings`, lifecycle `status`

**Node (discriminated union):**
- `AgentNode` -- AI agent step with instruction, model_provider, tool_refs, memory_refs, thread_participation
- `ExecutableUnitNode` -- Code/script step with manifest_ref, execution_mode, sandbox_config
- `HumanApprovalNode` -- Pause-for-human gate with approval payload schema
- All share `NodeBase`: node_id, graph_version_ref, input/output_contract_ref, policy_bindings, capability_bindings

**Run (execution instance):**
- Purpose: Mutable state of a single graph execution
- Location: `src/zeroth/runs/models.py`
- Extends GovernAI `RunState` with: pending_node_ids, execution_history, node_visit_counts, condition_results, audit_refs, failure_state, tenant_id, workspace_id, submitted_by

**Deployment (immutable snapshot):**
- Purpose: Production-ready frozen graph with pinned contract versions
- Location: `src/zeroth/deployments/models.py`
- Contains: serialized_graph, entry_input/output_contract_ref + version, snapshot digests, attestation_digest

**ServiceBootstrap (composition root):**
- Purpose: Wires all dependencies for a deployment-scoped service instance
- Location: `src/zeroth/service/bootstrap.py`
- Pattern: `@dataclass(slots=True)` holding all repositories, services, config; constructed by `bootstrap_service()` factory

## Entry Points

**FastAPI Application:**
- Location: `src/zeroth/service/app.py` -> `create_app(bootstrap)`
- Triggers: `bootstrap_app()` or direct `create_app()` call
- Responsibilities: HTTP routing, auth middleware, correlation IDs, lifespan (worker start/stop, queue gauge)

**Bootstrap Factory:**
- Location: `src/zeroth/service/bootstrap.py` -> `bootstrap_service(database, deployment_ref=...)`
- Triggers: Called with `SQLiteDatabase` and `deployment_ref` to wire a complete service
- Responsibilities: Load deployment, deserialize graph, verify integrity, construct all repositories/services

**RunWorker:**
- Location: `src/zeroth/dispatch/worker.py`
- Triggers: `worker.start()` + `worker.poll_loop()` in app lifespan
- Responsibilities: Orphan recovery, PENDING run polling, lease-based execution, metrics

## Error Handling

**Strategy:** Domain-specific exception hierarchies with structured error propagation

**Patterns:**
- Each module has its own error hierarchy: `OrchestratorError`, `AgentRuntimeError`, `ExecutableUnitError`, `GraphLifecycleError`, `ContractRegistryError`, `ManifestValidationError`, `SandboxPolicyViolationError`
- Orchestrator catches node execution errors, records audit for failures, marks run as `FAILED` with structured `RunFailureState(reason=..., message=...)`
- HTTP layer maps domain errors to status codes: `404` (not found), `409` (conflict), `422` (validation), `429` (rate limit), `503` (backpressure/quota)
- Policy violations produce `EnforcementResult` with `PolicyDecision.DENY` + reason string
- Auth failures recorded as audit events before returning 401/403

## Cross-Cutting Concerns

**Logging:** Standard Python `logging` module. Primary usage in `dispatch/worker.py` for operational events (claim, recovery, errors).

**Validation:** Pydantic `ConfigDict(extra="forbid")` on all models to reject unknown fields. Contract-based input validation on run creation. Graph structural validation. Manifest digest verification via `AdmissionController`.

**Authentication:** Two methods via `ServiceAuthenticator` in `src/zeroth/service/auth.py`:
- Static API keys: `X-API-Key` header matched against configured `StaticApiKeyCredential` list
- JWT bearer tokens: `Authorization: Bearer` with JWKS verification, issuer/audience validation
- Config loaded from env vars: `ZEROTH_SERVICE_API_KEYS_JSON`, `ZEROTH_SERVICE_BEARER_JSON`

**Authorization:** Role-based in `src/zeroth/service/authorization.py`:
- Three roles: `OPERATOR` (run create/read, approval read), `REVIEWER` (+ approval resolve), `ADMIN` (all permissions)
- Resource-scoped: `tenant_id` + `workspace_id` matching on deployments, runs, approvals
- Denials recorded as audit events

**Audit:** Append-only `NodeAuditRecord` for every node execution, policy decision, approval action, auth denial. Digest-chained via `previous_record_digest` / `record_digest` for tamper detection. Secret values redacted before persistence via `SecretRedactor`.

**Tenant Isolation:** Every `Run`, `Thread`, `Deployment`, and audit record carries `tenant_id` and optional `workspace_id`. Scope checks in authorization layer prevent cross-tenant data access. Scope mismatches return 404 (not 403) to hide resource existence.

**Correlation IDs:** Propagated via `X-Correlation-ID` header or auto-generated. Stored in context vars via `src/zeroth/observability/correlation.py`. Returned in response headers.

---

*Architecture analysis: 2026-04-05*
