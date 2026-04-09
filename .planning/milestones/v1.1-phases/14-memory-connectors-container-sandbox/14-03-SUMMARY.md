---
phase: 14-memory-connectors-container-sandbox
plan: 03
subsystem: memory
tags: [pgvector, chromadb, elasticsearch, vector-search, full-text-search, connectors]
dependency_graph:
  requires: [14-01]
  provides: [pgvector-connector, chroma-connector, elasticsearch-connector]
  affects: [memory-subsystem, agent-context-retrieval]
tech_stack:
  added: [pgvector, chromadb-client, elasticsearch-async, litellm]
  patterns: [connection-factory, protocol-compliance, upsert-semantics, scope-target-isolation]
key_files:
  created:
    - src/zeroth/memory/pgvector_connector.py
    - src/zeroth/memory/chroma_connector.py
    - src/zeroth/memory/elastic_connector.py
    - tests/memory/test_pgvector.py
    - tests/memory/test_chroma.py
    - tests/memory/test_elastic.py
  modified:
    - pyproject.toml
    - uv.lock
decisions:
  - PgvectorMemoryConnector uses conn_factory callable instead of conninfo string for flexible connection management
  - ChromaDB uses sync collection operations (HttpClient API is sync) - async embedding only
  - Elasticsearch uses NotFoundError exception for read/delete miss detection
metrics:
  duration: 285s
  completed: "2026-04-07"
  tasks_completed: 2
  tasks_total: 2
  tests_added: 29
  files_changed: 8
requirements: [MEM-03, MEM-04, MEM-05]
---

# Phase 14 Plan 03: Vector/Search Memory Connectors Summary

Three vector/search memory connectors (pgvector, ChromaDB, Elasticsearch) implementing GovernAI MemoryConnector async protocol with HNSW-indexed cosine similarity, HTTP client vector search, and full-text match queries respectively.

## What Was Done

### Task 1: PgvectorMemoryConnector
- Created `PgvectorMemoryConnector` with async psycopg connection factory pattern
- HNSW index with `vector_cosine_ops` for approximate nearest-neighbor search
- `embedding <=> %s` cosine similarity ordering in search queries
- Schema auto-creation: pgvector extension, table with vector column, HNSW index
- UPSERT semantics via `ON CONFLICT ... DO UPDATE`
- Embedding generation via `litellm.aembedding` with configurable model
- Scope+target isolation in all queries
- 10 unit tests passing (mocked psycopg + litellm)
- **Commit:** `06b9678`

### Task 2: ChromaDB and Elasticsearch Connectors
- Created `ChromaDBMemoryConnector` with HTTP client, collection-per-scope pattern
- ChromaDB collections configured with `{"hnsw:space": "cosine"}` metadata
- Created `ElasticsearchMemoryConnector` with async client, full-text match queries
- Elasticsearch uses `match_all` for empty text queries, `match` for text search
- Both connectors implement all 4 GovernAI MemoryConnector methods (read/write/delete/search)
- 19 unit tests passing (mocked chromadb + elasticsearch clients)
- **Commit:** `0a42f84`

## Dependencies Added
- `pgvector>=0.4.2` - PostgreSQL vector similarity search
- `litellm>=1.83.0` - Unified LLM API for embedding generation
- `psycopg[binary]>=3.1` - Async PostgreSQL adapter
- `chromadb-client>=1.5.6` - ChromaDB HTTP client
- `elasticsearch[async]>=8.0,<9` - Elasticsearch async client

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Missing litellm and psycopg dependencies**
- **Found during:** Task 1
- **Issue:** litellm and psycopg not in pyproject.toml dependencies
- **Fix:** Added via `uv add litellm "psycopg[binary]>=3.1"`
- **Files modified:** pyproject.toml, uv.lock
- **Commit:** 06b9678

**2. [Rule 1 - Bug] register_vector_async requires real psycopg connection in tests**
- **Found during:** Task 1
- **Issue:** `register_vector_async` rejects AsyncMock, needed patching in tests
- **Fix:** Patched `register_vector_async` in test fixture and set `_setup_done = True`
- **Files modified:** tests/memory/test_pgvector.py
- **Commit:** 06b9678

## Verification

All 29 unit tests pass, 3 live stubs skipped:
```
tests/memory/test_pgvector.py: 10 passed, 1 skipped
tests/memory/test_chroma.py: 9 passed, 1 skipped
tests/memory/test_elastic.py: 9 passed, 1 skipped
```

Ruff lint clean on all three connector modules.

## Known Stubs

None - all connectors are fully implemented with real logic (mocked only in tests).

## Self-Check: PASSED

- All 6 created files exist on disk
- Both task commits (06b9678, 0a42f84) verified in git log
- All 29 tests pass
