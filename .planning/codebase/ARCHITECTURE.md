# Zeroth Codebase Architecture

## High-Level Shape

Zeroth is organized as a modular domain-driven backend centered around workflow graphs, runtime execution, governance, and deployment-bound service APIs.

The architecture is not a monolith in one file tree. Instead, it is split into focused packages under [`src/zeroth/`](`src/zeroth/`) with a thin HTTP surface that wires repositories and services together per deployment.

## Main Architectural Pattern

The dominant pattern is:

- domain models and validators
- repository-backed persistence
- domain services
- deployment-scoped HTTP bootstrapping

This is visible in packages such as:

- `graph`
- `contracts`
- `runs`
- `deployments`
- `approvals`
- `audit`
- `dispatch`
- `service`

## Core Execution Flow

At a high level:

1. A graph is defined, validated, versioned, and published through `src/zeroth/graph/`
2. A deployment snapshot is created via `src/zeroth/deployments/`
3. A deployment-bound FastAPI service is bootstrapped in [`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`)
4. HTTP routes are registered in [`src/zeroth/service/app.py`](`src/zeroth/service/app.py`)
5. Runs are created through [`src/zeroth/service/run_api.py`](`src/zeroth/service/run_api.py`)
6. A durable worker in `src/zeroth/dispatch/` polls pending runs and hands them to the runtime orchestrator
7. The orchestrator in [`src/zeroth/orchestrator/runtime.py`](`src/zeroth/orchestrator/runtime.py`) coordinates agents, executable units, conditions, approvals, audits, and run state transitions

## Entry Points

Primary backend entry points include:

- [`src/zeroth/service/app.py`](`src/zeroth/service/app.py`) — FastAPI app factory
- [`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`) — deployment-scoped dependency wiring
- [`src/zeroth/orchestrator/runtime.py`](`src/zeroth/orchestrator/runtime.py`) — runtime coordination core

There is no separate UI/application shell entry point in the current repo.

## Package Responsibilities

- `graph` owns authorable/publishable graph shape, storage, validation, diffs, and versioning
- `contracts` owns typed data contract registration and model resolution
- `mappings` owns transformation of payloads between node boundaries
- `agent_runtime` owns prompt assembly, providers, tool wiring, and thread state hooks
- `execution_units` owns executable-unit manifests, adapters, IO, sandboxing, and integrity
- `conditions` owns conditional logic and branch resolution
- `runs` owns run/thread persistence and lifecycle state
- `approvals` owns paused-for-human-review workflows
- `audit` owns evidence, verifier, timeline, and sanitized audit access
- `deployments` owns graph deployment snapshots and provenance
- `dispatch` owns worker leasing and recovery behavior
- `guardrails` owns quotas, rate limiting, and dead-letter logic
- `observability` owns correlation IDs, queue gauges, and metrics
- `service` exposes the deployment-bound HTTP APIs

## Wiring Style

The code uses explicit object construction rather than a heavy dependency injection framework.

Examples:

- repositories are instantiated directly in [`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`)
- `ServiceBootstrap` collects the deployment-scoped dependencies
- the FastAPI app stores bootstrap state on `app.state.bootstrap`

This keeps runtime composition straightforward and testable, though it can create a large bootstrap surface as the system grows.

## Data Flow Characteristics

- contracts validate payloads at boundaries
- mappings transform payloads between nodes
- run metadata persists workflow state and execution progress
- audit and evidence records capture governance details
- approvals pause and resume execution through persisted state

The architecture is stateful and persistence-aware; it is not a stateless request/response service only.

## Notable Architectural Boundary

The current service app is deployment-bound, not workspace-wide. A single bootstrap loads one deployment snapshot and exposes routes for that deployment context. This is important because a future Studio layer likely needs a broader authoring/control-plane boundary than the current deployment wrapper provides.

## Current Architectural Gap Relevant To Studio

There is no dedicated authoring/control-plane package yet for:

- draft workflow CRUD
- revision editing leases
- authoring-time assets
- environments registry
- Studio-oriented session/bootstrap APIs

Those responsibilities will likely need a new package, such as `src/zeroth/studio/`, instead of being squeezed into the existing deployment service wrapper.
