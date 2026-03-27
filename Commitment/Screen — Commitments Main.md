**File Name**: screen_commitments **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** portfolio overview of all commitments. Shows each commitment's name, recurrence, and current state at a glance. Not for logging — for reviewing, managing, and navigating to details.

---

## Layout

```
┌─────────────────────────────────────────┐
│  MY COMMITMENTS                         │
│                                         │
│  [ Active ] [ Frozen ] [ Completed ]    │
│                                         │
│  DO ──────────────────── 3              │
│  Morning Walk       daily               │
│  Read 30 pages      weekly              │
│  Exercise 3x        weekly              │
│                                         │
│  AVOID ──────────────── 2               │
│  Coffee ≤1          daily               │
│  No alcohol         daily               │
│                                         │
│              [ + Add ]                  │
└─────────────────────────────────────────┘
```

---

## Commitment Rows

Grouped by commitment type — Do first, Avoid below. Each group header shows the type label and count. Tapping a group header collapses or expands the group.

Each row is a Commitment Card. See `component_commitment_card` for the card's layout, states, and gestures. The screen provides each card with its commitment data from the provider — the card renders it.

The filter chips control which commitment stream feeds the list:

- Active → cards from `watchActiveCommitments()`
- Frozen → cards from `watchFrozenCommitments()`
- Completed → cards from `watchCompletedCommitments()`

---

## Filter Chips

`[ Active ] [ Frozen ] [ Completed ]` — Active selected by default.

Deleted commitments are not shown here. They are accessible via Settings → Recycle Bin. See `component_recycle_bin`.

---

## Add Button

Prominent, always visible at the bottom. Behavior handled by the Add Commitment Button component. See `component_add_commitment_button`.

---

## Navigation

- Tap row → Commitment Detail screen
- Long press → Edit → Commitment Form (edit mode)
- Add button → Commitment Form (create mode)
- Settings → Recycle Bin → deleted commitments

---

## Components

|Component|Location|Doc|
|---|---|---|
|Commitment Card|Each row|`component_commitment_card`|
|Add Commitment Button|Bottom of screen|`component_add_commitment_button`|

---

## Data Sources

|Data|Source|
|---|---|
|Active commitments|`CommitmentService.watchActiveCommitments()` — stream|
|Frozen commitments|`CommitmentService.watchFrozenCommitments()` — stream|
|Completed commitments|`CommitmentService.watchCompletedCommitments()` — stream|

---

## Later Improvements

These are not in scope for Phase 1. Documented here so the intent is not lost.

**Status icons per row.** Shows current window state at a glance — ✓ complete, ⚠ window closing, ✕ not a scheduled day. Requires `CommitmentIdentityService.watchInstancesForDay(today)` added to the screen's provider.

**Progress bar per row.** A visual fill showing `livePerformance` for the current window. Requires the Performance feature and the instance stream above.

**Group average score.** A summary score per Do / Avoid group in the group header. Requires `PerformanceService.getPerformanceForPeriod()` scoped to each group's commitments.

**Slots teaser.** A summary row above the Add button showing active commitment count and 7-day consistency, with a link to the Your Record screen. Requires the Performance feature for the consistency value and the Commitment Health component for the dot display.

**Commitment Health dots.** One colored dot per commitment reflecting current window performance. Requires `PerformanceService` and the `component_commitment_health` component.