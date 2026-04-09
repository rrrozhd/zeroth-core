---
phase: 12-real-llm-providers-retry
plan: 02
subsystem: api
tags: [retry, exponential-backoff, jitter, litellm, error-classification]

requires:
  - phase: 12-real-llm-providers-retry
    provides: LiteLLM provider adapter foundation (plan 01)
provides:
  - Retry utility module with transient error classification
  - Exponential backoff with jitter for provider calls
  - Enhanced RetryPolicy with base_delay/max_delay/use_exponential_backoff
affects: [12-real-llm-providers-retry, agent-runtime, provider-integration]

tech-stack:
  added: []
  patterns: [exponential-backoff-with-full-jitter, transient-error-classification]

key-files:
  created:
    - src/zeroth/agent_runtime/retry.py
  modified:
    - src/zeroth/agent_runtime/models.py
    - src/zeroth/agent_runtime/runner.py

key-decisions:
  - "Full jitter strategy (uniform 0..delay) chosen over decorrelated jitter for simplicity"
  - "Exponential backoff enabled by default (use_exponential_backoff=True) with backoff_seconds retained as fallback"
  - "Error classification uses litellm exception types with status_code fallback for unknown exceptions"

patterns-established:
  - "Transient error classification: check litellm exception types then fall back to status_code attribute"
  - "Backoff computation separated from retry loop for testability"

requirements-completed: [LLM-03]

duration: 3min
completed: 2026-04-06
---

# Phase 12 Plan 02: Retry & Backoff Summary

**Exponential backoff with jitter and transient error classification for LLM provider retry loop**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-06T21:41:01Z
- **Completed:** 2026-04-06T21:43:38Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Created retry.py module with is_retryable_provider_error (classifies litellm exceptions) and compute_backoff_delay (exponential backoff with full jitter)
- Enhanced RetryPolicy model with base_delay, max_delay, and use_exponential_backoff fields while preserving backward compatibility
- Upgraded AgentRunner retry loop to use exponential backoff for transient errors and fail immediately on permanent errors (401, 400, 403)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create retry module with error classification** - `0fa91a1` (feat)
2. **Task 2: Upgrade runner retry loop to exponential backoff** - `88614bf` (feat)

## Files Created/Modified
- `src/zeroth/agent_runtime/retry.py` - Error classification (is_retryable_provider_error) and backoff computation (compute_backoff_delay)
- `src/zeroth/agent_runtime/models.py` - RetryPolicy enhanced with base_delay, max_delay, use_exponential_backoff fields
- `src/zeroth/agent_runtime/runner.py` - Retry loop uses exponential backoff for transient errors, permanent errors fail immediately

## Decisions Made
- Full jitter strategy (uniform random between 0 and computed delay) for backoff -- simplest effective approach per AWS architecture blog patterns
- Exponential backoff enabled by default; existing code unaffected since default base_delay=1.0 only applies when retries > 0
- backoff_seconds field retained for backward compatibility when use_exponential_backoff=False

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed ruff SIM103 lint violation in retry.py**
- **Found during:** Task 1 (verification)
- **Issue:** ruff flagged return True/return False pattern as SIM103 (should inline condition)
- **Fix:** Replaced if/return True/return False with direct return of boolean expression
- **Files modified:** src/zeroth/agent_runtime/retry.py
- **Verification:** uv run ruff check passes
- **Committed in:** 0fa91a1 (Task 1 commit)

**2. [Rule 1 - Bug] Fixed import ordering in runner.py**
- **Found during:** Task 2 (verification)
- **Issue:** ruff flagged I001 unsorted import block after adding retry import
- **Fix:** Moved retry import to alphabetically correct position among zeroth.agent_runtime imports
- **Files modified:** src/zeroth/agent_runtime/runner.py
- **Verification:** uv run ruff check passes
- **Committed in:** 88614bf (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (2 bugs/lint)
**Impact on plan:** Minor lint fixes only. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Retry module ready for plan 03 (model fallback chain)
- All 280 existing tests pass with no regressions
- RetryPolicy backward compatible -- existing configs work without changes

---
*Phase: 12-real-llm-providers-retry*
*Completed: 2026-04-06*
