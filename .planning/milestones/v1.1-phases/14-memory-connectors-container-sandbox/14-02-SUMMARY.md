---
phase: 14-memory-connectors-container-sandbox
plan: 02
subsystem: memory
tags: [redis, memory-connector, governai-protocol, persistence]
dependency_graph:
  requires: [14-01]
  provides: [RedisKVMemoryConnector, RedisThreadMemoryConnector]
  affects: [14-05]
tech_stack:
  added: [redis.asyncio]
  patterns: [sorted-set-history, namespaced-redis-keys, async-connector-protocol]
key_files:
  created:
    - src/zeroth/memory/redis_kv.py
    - src/zeroth/memory/redis_thread.py
    - tests/memory/test_redis_kv.py
    - tests/memory/test_redis_thread.py
  modified: []
decisions:
  - Upsert semantics for KV write preserving created_at timestamp on updates
  - Thread connector uses ZADD with float timestamp scores for natural ordering
  - search() scans all matching keys via SCAN_ITER rather than maintaining a secondary index
metrics:
  duration: 215s
  completed: "2026-04-07T08:42:34Z"
  tasks: 2
  files: 4
  tests_added: 28
---

# Phase 14 Plan 02: Redis KV and Thread Memory Connectors Summary

Redis-backed key-value and sorted-set thread connectors implementing GovernAI MemoryConnector protocol with namespaced key isolation and async operations.

## What Was Built

### Task 1: RedisKVMemoryConnector
- **Commit:** cfec02f
- Created `src/zeroth/memory/redis_kv.py` with `RedisKVMemoryConnector` class
- Implements all 4 GovernAI MemoryConnector methods: `read`, `write`, `delete`, `search`
- Uses Redis GET/SET/DEL/SCAN_ITER for key-value persistence
- Key format: `{prefix}:{scope}:{target}:{key}` (e.g., `zeroth:mem:kv:run:run-1:user_prefs`)
- Upsert semantics: preserves `created_at` on updates, updates `updated_at`
- 15 unit tests covering protocol conformance, CRUD, search, scope isolation, key format

### Task 2: RedisThreadMemoryConnector
- **Commit:** 1732485
- Created `src/zeroth/memory/redis_thread.py` with `RedisThreadMemoryConnector` class
- Implements all 4 GovernAI MemoryConnector methods using Redis sorted sets
- ZADD with timestamp scores for append-only conversation history
- ZREVRANGE for reading most recent entry
- Key format: `{prefix}:{scope}:{target}:{key}` (e.g., `zeroth:mem:thread:thread:t-1:messages`)
- Search supports `text` and `limit` query parameters
- 13 unit tests covering protocol conformance, append behavior, search, isolation

## Key Design Decisions

1. **Distinct key prefixes** (`zeroth:mem:kv` vs `zeroth:mem:thread`) prevent data collision between connector types
2. **Sorted set scores** use `time.time()` float for natural chronological ordering
3. **TYPE_CHECKING guard** for `redis.asyncio` import avoids runtime import overhead when type-checking only
4. **Search via SCAN_ITER** trades index maintenance complexity for simplicity; acceptable for MVP with future optimization path

## Deviations from Plan

None -- plan executed exactly as written.

## Verification

- 28 tests pass: `uv run pytest tests/memory/test_redis_kv.py tests/memory/test_redis_thread.py -x -v`
- Ruff clean: `uv run ruff check src/zeroth/memory/redis_kv.py src/zeroth/memory/redis_thread.py`
- Both connectors pass `isinstance(connector, MemoryConnector)` runtime check
- Prefixes are distinct (no collision)
- All operations use `redis.asyncio` (no sync calls)

## Known Stubs

None -- all functionality is fully wired.

## Self-Check: PASSED

- [x] src/zeroth/memory/redis_kv.py exists
- [x] src/zeroth/memory/redis_thread.py exists
- [x] tests/memory/test_redis_kv.py exists
- [x] tests/memory/test_redis_thread.py exists
- [x] Commit cfec02f found
- [x] Commit 1732485 found
