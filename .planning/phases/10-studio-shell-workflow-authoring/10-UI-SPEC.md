---
phase: 10
slug: studio-shell-workflow-authoring
status: approved
shadcn_initialized: false
preset: none
created: 2026-03-30
reviewed_at: 2026-03-30T11:55:00+03:00
---

# Phase 10 — UI Design Contract

> Visual and interaction contract for frontend phases. Generated for the initial Zeroth Studio shell and workflow authoring foundation.

---

## Design System

| Property | Value |
|----------|-------|
| Tool | none |
| Preset | not applicable |
| Component library | none |
| Icon library | lucide-vue-next |
| Font | Instrument Sans |

---

## Spacing Scale

Declared values (must be multiples of 4):

| Token | Value | Usage |
|-------|-------|-------|
| xs | 4px | Icon gaps, inline padding, badge offsets |
| sm | 8px | Compact control spacing, list row breathing room |
| md | 16px | Default control spacing, inspector padding |
| lg | 24px | Panel padding, grouped control spacing |
| xl | 32px | Major layout gaps, shell spacing |
| 2xl | 48px | Section breaks inside full-screen views |
| 3xl | 64px | Page-level vertical spacing for non-canvas screens |

Exceptions: none

---

## Typography

| Role | Size | Weight | Line Height |
|------|------|--------|-------------|
| Body | 14px | 400 | 1.5 |
| Label | 12px | 600 | 1.3 |
| Heading | 20px | 600 | 1.2 |
| Display | 28px | 600 | 1.1 |

---

## Color

| Role | Value | Usage |
|------|-------|-------|
| Dominant (60%) | #F4F7F8 | App background, canvas shell backdrop, primary surfaces |
| Secondary (30%) | #E6ECEF | Workflow rail, inspector background, segmented controls, quiet panels |
| Accent (10%) | #0F766E | Primary CTA, active mode tab, selected workflow row, focus ring, selected node outline |
| Destructive | #C2410C | Delete, remove, destructive confirmations only |

Accent reserved for: primary CTA, active mode switch state, focused/selected workflow row, selected node outline, keyboard focus ring

---

## Copywriting Contract

| Element | Copy |
|---------|------|
| Primary CTA | Run Draft |
| Empty state heading | Start your first workflow |
| Empty state body | Add an agent or executable unit to begin shaping the workflow. |
| Error state | Studio couldn't save this draft. Check the validation hints, then try saving again. |
| Destructive confirmation | Delete workflow: This removes the draft and its unpublished changes. |

---

## Visual Hierarchy Contract

- Primary focal point: the center canvas and the currently selected workflow name in the header
- Secondary focal point: the active top mode switch state (`Editor`, `Executions`, or `Tests`)
- Tertiary focal point: the current node selection in the right inspector
- The left workflow rail must read as quiet navigation, not as the dominant visual plane
- Runtime status indicators must stay ambient and compact inside Editor mode
- Icon-only actions require tooltip or visible text fallback on hover/focus

---

## Interaction Contract

- Default shell posture is canvas-first
- Left rail is workflows-only, organized as folders and rows rather than card grids
- `Assets` appears as a secondary utility entry, not a peer to the main workflow list
- Right inspector opens on selection and stays narrow, focused, and contextual
- Runtime detail in Editor mode is limited to compact recent activity
- Deeper run and governance inspection belongs behind explicit navigation into `Executions`
- Header environment control uses a dropdown with a `Manage environments` action

---

## Registry Safety

| Registry | Blocks Used | Safety Gate |
|----------|-------------|-------------|
| shadcn official | none | not required |
| third-party registries | none | not applicable |

---

## Checker Sign-Off

- [x] Dimension 1 Copywriting: PASS
- [x] Dimension 2 Visuals: PASS
- [x] Dimension 3 Color: PASS
- [x] Dimension 4 Typography: PASS
- [x] Dimension 5 Spacing: PASS
- [x] Dimension 6 Registry Safety: PASS

**Approval:** approved 2026-03-30
