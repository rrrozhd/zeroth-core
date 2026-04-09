---
phase: 22-canvas-foundation-dev-infrastructure
plan: 02
subsystem: studio-api
tags: [api, rest, crud, studio, graph-authoring]
dependency_graph:
  requires: [graph-repository, graph-models, service-app]
  provides: [studio-api-router, studio-schemas, node-type-registry]
  affects: [studio-frontend, canvas-integration]
tech_stack:
  added: []
  patterns: [visual-metadata-in-graph-metadata, soft-delete-via-archive, static-node-type-registry]
key_files:
  created:
    - src/zeroth/service/studio_api.py
    - src/zeroth/service/studio_schemas.py
  modified:
    - src/zeroth/service/app.py
    - tests/test_studio_api.py
decisions:
  - Visual metadata stored in graph.metadata["studio"] to avoid modifying core models
  - Soft delete via archive status transition rather than physical deletion
  - 8 frontend visual types mapped to 3 backend execution types via studio layer
metrics:
  duration: 160s
  completed: "2026-04-09T13:58:56Z"
  tasks: 1
  files: 4
---

# Phase 22 Plan 02: Studio Graph Authoring API Summary

Studio REST API at /api/studio/v1/ with CRUD workflow endpoints and node-type registry, storing visual metadata (positions, viewport) in graph.metadata["studio"] without modifying core graph models.

## What Was Built

### Studio API Router (src/zeroth/service/studio_api.py)

FastAPI router mounted at `/api/studio/v1/` with 6 endpoints:

1. **GET /workflows** -- List all non-archived workflows as summaries
2. **POST /workflows** -- Create a new workflow with default studio metadata (201)
3. **GET /workflows/{id}** -- Get full workflow detail with nodes, edges, viewport
4. **PUT /workflows/{id}** -- Update workflow name, nodes, edges, or viewport
5. **DELETE /workflows/{id}** -- Soft delete via archive status transition (204)
6. **GET /node-types** -- Return 8 static node type definitions with port configs

### Studio Schemas (src/zeroth/service/studio_schemas.py)

Pydantic models for the Studio API:
- `CreateWorkflowRequest`, `UpdateWorkflowRequest` -- request bodies
- `WorkflowSummaryResponse`, `WorkflowDetailResponse` -- response models
- `StudioNodeResponse`, `StudioEdgeResponse` -- node/edge representations
- `StudioPosition`, `StudioViewport` -- canvas state models
- `NodeTypeResponse`, `PortDefinitionResponse` -- node type registry models

### Node Type Registry

8 frontend visual types with port definitions:
- **flow**: start (output-control), end (input-control), conditionBranch (input-data, output-true, output-false)
- **core**: agent (input/output-data), executionUnit (input/output-data), approvalGate (input/output-control)
- **data**: memoryResource (input/output-data), dataMapping (input/output-data)

### App Integration

Studio router mounted in `create_app()` -- existing auth middleware automatically covers all Studio routes.

## Decisions Made

1. **Visual metadata in graph.metadata["studio"]** -- Node positions and viewport state stored as nested dict in the graph's metadata field. This avoids modifying the core Graph model (which has extra="forbid") while keeping visual state persisted with the graph.

2. **Soft delete via archive** -- DELETE endpoint transitions workflow to ARCHIVED status using the existing `Graph.archive()` method. List endpoint filters out archived workflows. This preserves audit trail.

3. **Static node type registry** -- The 8 frontend visual types are defined as a static list in the router module. No database access needed. This maps to n8n's approach of declaring node types in code.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] SQLite in-memory database incompatible with multi-connection access**
- **Found during:** Task 1 (TDD GREEN)
- **Issue:** Using `SQLiteDatabase(":memory:")` in tests created a new empty database for each `transaction()` call since each call opens a new connection.
- **Fix:** Changed test helper to use file-based temp SQLite databases instead of `:memory:`
- **Files modified:** tests/test_studio_api.py
- **Commit:** e8fc051

## Test Results

10 tests, all passing:
- TestCreateWorkflow: create (201), empty name rejected (422)
- TestListWorkflows: list returns summaries
- TestGetWorkflow: get detail, not found (404)
- TestUpdateWorkflow: update name, not found (404)
- TestDeleteWorkflow: delete (204), not found (404)
- TestListNodeTypes: 8 types with ports

## Commits

| Hash | Type | Description |
|------|------|-------------|
| d6cc939 | test | Add failing tests for Studio API CRUD and node-types |
| e8fc051 | feat | Implement Studio API with CRUD workflows and node-types registry |

## Known Stubs

None -- all endpoints are fully wired to the GraphRepository.

## Self-Check: PASSED

- All 3 created/modified source files exist
- Both commits (d6cc939, e8fc051) verified in git log
- All 16 acceptance criteria confirmed (grep counts > 0)
- 10/10 tests passing
- Lint clean (ruff check passes)
