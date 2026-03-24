**File Name**: screen_commitments **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 23-Mar-2026

---

**Purpose:** portfolio overview of all commitments. Shows each commitment's current state at a glance. Not for logging — for reviewing, managing, and navigating to details.

---

## Layout

```
┌─────────────────────────────────────────┐
│  MY COMMITMENTS                         │
│                                         │
│  [ Active ] [ Frozen ] [ Completed ]    │
│                                         │
│  DO ──────────────────── 3              │
│  Morning Walk    daily      ████████ ✓  │
│  Read 30 pages   weekly     ████░░░░ ⚠  │
│  Exercise 3x     weekly     ████████ ✓  │
│                                         │
│  AVOID ──────────────── 2               │
│  Coffee ≤1       daily      ██████░░ ⚠  │
│  No alcohol      daily      ████████ ✓  │
│                                         │
│              [ + Add ]                  │
└─────────────────────────────────────────┘
```

---

## Commitment Rows

Grouped by commitment type — Do first, Avoid below. Each group header shows the type label and count. Tapping a group header collapses or expands the group.

Each row shows the commitment name, recurrence label, a progress bar reflecting `livePerformance`, and a status icon. Raw percentage scores are not shown — the progress bar and icon carry that meaning. See `ux_principles` rule 10.

Status icons: ✓ at 100%, ⚠ when the activity window is closing (3/4 elapsed), no icon otherwise.

---

## Filter Chips

`[ Active ] [ Frozen ] [ Completed ]` — Active selected by default.

Deleted commitments are not shown here. They are accessible via Settings → Recycle Bin. See `recycle_bin` component.

---

## Gestures on Each Row

Active rows:

- **Tap** → Commitment Detail screen
- **Swipe left** → soft delete — moves commitment to deleted state silently, no confirmation dialog. Commitment is recoverable from Settings → Recycle Bin.
- **Long press** → action menu: Edit · Freeze · Delete

Frozen and completed rows: tap only — no swipe gestures.

Soft delete is silent by design. The commitment is not gone — it is in the Recycle Bin. A brief undo snackbar may appear immediately after the swipe to offer a quick reversal before the user navigates away.

---

## Add Button

Prominent, always visible at the bottom of the screen. Behavior is handled entirely by the Add Commitment Button component — the screen just places it. See `component_add_commitment_button`.

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
|Commitment Card (threads variant)|Each row|`commitment_card`|
|Add Commitment Button|Bottom of screen|`component_add_commitment_button`|

---

## Data Sources

| Data                      | Source                                                           |
| ------------------------- | ---------------------------------------------------------------- |
| Active commitments        | `CommitmentService.watchActiveCommitments()` — stream            |
| Frozen commitments        | `CommitmentService.watchFrozenCommitments()` — stream            |
| Completed commitments     | `CommitmentService.watchCompletedCommitments()` — stream         |
| Today's instance progress | `CommitmentIdentityService.watchInstancesForDay(today)` — stream |