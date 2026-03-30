---
phase: 10-studio-shell-workflow-authoring
plan: 04
subsystem: ui
tags: [vue, vite, pinia, vue-router, vue-query, vue-flow, studio-shell]
requires:
  - phase: 10-02
    provides: "Typed Studio workflow and lease APIs used by the shell data layer"
  - phase: 10-03
    provides: "Frontend workspace, router bootstrap, shell tokens, and Studio API client"
provides:
  - "Canvas-first Studio shell rooted in AppShell"
  - "Workflow-first rail grouped by folder_path with a separate Assets utility row"
  - "Header, mode switch, dominant canvas, and contextual inspector baseline for editor/executions/tests"
affects: [phase-10-plan-06, phase-11-runtime-views, studio-frontend]
tech-stack:
  added: []
  patterns: ["Vue route-per-mode shell composition", "TanStack Query fallback loading for Studio frontend shells"]
key-files:
  created:
    - apps/studio/src/app/AppShell.vue
    - apps/studio/src/features/workflows/WorkflowRail.vue
    - apps/studio/src/features/canvas/WorkflowCanvas.vue
    - apps/studio/src/features/header/StudioHeader.vue
    - apps/studio/src/features/modes/ModeSwitch.vue
    - apps/studio/src/features/inspector/NodeInspector.vue
  modified:
    - apps/studio/src/router/index.ts
    - apps/studio/src/main.ts
    - apps/studio/src/stores/studioShell.ts
    - apps/studio/src/lib/api/studio.ts
    - apps/studio/src/styles/tokens.css
key-decisions:
  - "Used route-level editor/executions/tests entries that each mount the same AppShell so the shell chrome stays shared while the mode state remains URL-driven."
  - "Added local fallback workflow summaries and graph data so the shell still renders meaningfully when the Studio backend is unavailable."
  - "Kept the inspector intentionally placeholder-oriented and compact rather than introducing runtime tables or asset editors in this plan."
patterns-established:
  - "Studio shell components live under apps/studio/src/features/* and compose through a thin AppShell root."
  - "Workflow rail and canvas both hydrate through the typed Studio API client but degrade gracefully to local authoring-safe fallback data."
requirements-completed: [STU-01, UX-01, UX-02]
duration: 17 min
completed: 2026-03-30
---

# Phase 10 Plan 04: Studio Shell Layout Summary

**Canvas-first Studio shell with foldered workflow navigation, shared editor/executions/tests chrome, and a dominant Vue Flow authoring surface**

## Performance

- **Duration:** 17 min
- **Started:** 2026-03-30T14:06:00Z
- **Completed:** 2026-03-30T14:23:43Z
- **Tasks:** 1
- **Files modified:** 13

## Accomplishments

- Replaced the route skeleton with a real `AppShell` that composes header, rail, mode switch, canvas, and inspector.
- Implemented a quiet workflow rail that groups backend workflow summaries by `folder_path` and keeps `Assets` as a separate utility entry.
- Mounted `@vue-flow/core` as the central editor surface and kept non-editor modes contextual instead of turning the shell into an operations dashboard.

## Task Commits

Each task was committed atomically:

1. **Task 1: Build the shell layout with workflow rail, mode switch, canvas, and inspector** - `57b1045` (feat)

## Files Created/Modified

- `apps/studio/src/app/AppShell.vue` - Shared shell composition and route-mode synchronization.
- `apps/studio/src/features/workflows/WorkflowRail.vue` - Foldered workflow navigation, selection state, and `Assets` utility row.
- `apps/studio/src/features/canvas/WorkflowCanvas.vue` - Vue Flow canvas with selected-workflow loading and mode-aware center-panel copy.
- `apps/studio/src/features/header/StudioHeader.vue` - Workflow title, save state, environment control, and `Run Draft` or `Publish` actions.
- `apps/studio/src/features/modes/ModeSwitch.vue` - Compact segmented control for `Editor`, `Executions`, and `Tests`.
- `apps/studio/src/features/inspector/NodeInspector.vue` - Narrow contextual inspector with contract and recent-activity placeholders.
- `apps/studio/src/router/index.ts` - URL-driven mode routes that mount the shell directly.
- `apps/studio/src/main.ts` - Router-root bootstrap for the Studio shell app.
- `apps/studio/src/stores/studioShell.ts` - Shell state for workflow selection, environment label, and save status.
- `apps/studio/src/lib/api/studio.ts` - Workflow detail contract updated with `last_saved_at`.
- `apps/studio/src/styles/tokens.css` - Full shell layout, panel, and canvas styling aligned to the approved UI contract.

## Decisions Made

- Used route-level shell mounting instead of nested views because this plan only needs URL-stable mode boundaries, not separate mode components yet.
- Preserved a quiet shell by keeping runtime or governance copy ambient in the center card and inspector rather than adding deeper data tables.
- Allowed local fallback data for workflows and graph nodes so the frontend remains usable during isolated shell development and build verification.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Restored missing Studio frontend dependencies before verification**
- **Found during:** Task 1 (Build the shell layout with workflow rail, mode switch, canvas, and inspector)
- **Issue:** `npm --prefix apps/studio run build` failed because `apps/studio/node_modules` was missing and `vue-tsc` was unavailable.
- **Fix:** Ran `npm --prefix apps/studio ci` to restore the locked frontend toolchain, then reran the required build.
- **Files modified:** none committed
- **Verification:** `npm --prefix apps/studio run build`
- **Committed in:** `57b1045` (task commit; artifact captured after verification)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** The fix was environment-only and required to execute the planned build verification. No product scope changed.

## Issues Encountered

- `Background` was imported from `@vue-flow/core`, but that package version does not export it. The shell kept its visual depth in CSS and the Vue Flow canvas remained buildable without adding another dependency.

## Known Stubs

- `apps/studio/src/features/workflows/WorkflowRail.vue:12` uses local fallback workflow summaries when the backend query fails so the rail stays populated during isolated frontend work.
- `apps/studio/src/features/canvas/WorkflowCanvas.vue:17` uses a local fallback workflow graph when the selected workflow cannot be loaded from the backend.
- `apps/studio/src/features/inspector/NodeInspector.vue:24` hardcodes compact recent-activity rows and contract summary copy until node-local runtime data arrives in later plans.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Plan 10-06 can build lease-aware autosave and node-local validation on top of a stable shell structure and selected-workflow state.
- Phase 11 can replace the center-pane copy with real executions and tests views without reworking the outer shell composition.

## Self-Check: PASSED

- Verified `.planning/phases/10-studio-shell-workflow-authoring/10-04-SUMMARY.md` exists.
- Verified task commit `57b1045` is present in git history.

---
*Phase: 10-studio-shell-workflow-authoring*
*Completed: 2026-03-30*
