**File Name**: screen_commitment_detail **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** shows the full details of one commitment and provides access to logging. Minimal in Phase 1 — commitment facts, today's progress, and two action buttons. Accessible from both the Dashboard and the Commitments screen.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  Morning Walk                    ⋮  │
│                                         │
│  [ Commitment Info ]                    │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│   [ + Log ]       [ View Log History ] │
└─────────────────────────────────────────┘
```

---

## What This Screen Shows

The commitment's facts and today's progress, rendered by the Commitment Info component. See `component_commitment_info`. The screen provides the data — the component renders it.

---

## ⋮ Menu

Handled entirely by the Commitment State Change component. The detail screen places the component in the top-right corner — the component owns the menu, the dialogs, and the service calls. See `component_commitment_state_change`.

---

## Action Buttons

Two buttons, always visible at the bottom.

**+ Log** — opens the log entry dialog for this commitment. See `component_log_entry_dialog`.

**View Log History** — opens the log history sheet showing all past log entries for this commitment. See `component_log_history_sheet`.

---

## Components

|Component|Location|Doc|
|---|---|---|
|Commitment Info|Below app bar|`component_commitment_info`|
|Commitment State Change|Top-right ⋮ menu|`component_commitment_state_change`|

---

## Entry Points

No behavioral difference between entry points:

- Dashboard → tap commitment card
- Commitments screen → tap commitment row

---

## Navigation

- Back → returns to previous screen
- Edit (⋮ menu) → Commitment Form (edit mode) — via `component_commitment_state_change`
- - Log → log entry dialog
- View Log History → log history sheet

---

## Data Sources

|Data|Source|
|---|---|
|Commitment definition|`CommitmentService.getDefinition(id)` — one-time read|
|Today's instance (progress, window state)|`CommitmentIdentityService.watchInstancesForDay(today)` — stream|

---

## Later Improvements

These are not in scope for Phase 1. Documented here so the intent is not lost.

**Performance history.** A weekly breakdown of how the user has performed on this commitment over time — scores per week, trend direction. Requires `PerformanceService.getCommitmentWeekScore()`.

**Garment display.** The garment visual showing long-term weaving progress for this commitment. Replaces the plain text facts at the top as the primary visual. Requires the Garment feature (Phase 2).

**Weekly progress table.** A scrollable list of weekly garment deltas — how much the garment grew or decayed each week. Shown alongside the garment display (Phase 2).

**Micro-insight.** An AI-generated pattern observation shown when the commitment's 7-day score is below 50%. Pro and Premium users only. Requires the AI Insights feature (Phase 3).

**Scheduled freeze summary.** A display of which days are auto-frozen for this commitment. See `component_scheduled_freeze`.