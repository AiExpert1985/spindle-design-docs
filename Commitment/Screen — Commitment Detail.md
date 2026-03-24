**File Name**: screen_commitment_detail **Feature**: Commitment **Phase**: 1 (core) · Phase 2 (garment + weekly progress) · Phase 3 (AI insight) **Created**: 15-Mar-2026 **Modified**: 23-Mar-2026

---

**Purpose:** deep view of one commitment. Shows current state, garment progress, weekly history, and AI insights. Built incrementally — components are added per phase. Accessible from both the Dashboard and the Commitments screen.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  Morning Walk                    ⋮  │
│                                         │
│  [ Score Ring ]               Phase 1   │  ← replaced by Garment Display in Phase 2
│                                         │
│  [ Commitment Info ]          Phase 1   │
│  do · daily · 5km · 6–10am             │
│  3 of 5km today · 7 day streak 🥇      │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  [ Garment Display ]          Phase 2   │  ← replaces Score Ring
│    62% woven                            │
│    This week: +3% so far    ↑           │
│                                         │
│  WEEKLY PROGRESS              Phase 2   │
│  This week  (Mon–today)  +3%  ███░░     │
│  Week of Mar 10          +5%  █████░    │
│  Week of Mar 3           +3%  ███░░     │
│  Week of Feb 24          -1%  decay     │
│  Week of Feb 17          +7%  ███████░  │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  [ Understand this pattern ]  Phase 3   │
│                                         │
│              [ + Log ]                  │
└─────────────────────────────────────────┘
```

---

## Phase Transitions

**Phase 1:** Score Ring and Commitment Info only. Functional but minimal. The ring shows today's `livePerformance` for this commitment. Weekly history is absent — acceptable for internal testing.

**Phase 2:** Score Ring is removed. Garment Display takes its place as the primary visual. Weekly Progress table added below. The garment completion percentage is shown as a number below the garment — the garment carries the feeling, the number carries the precision.

**Phase 3:** Micro-Insight section added at the bottom for Pro and Premium users when the 7-day score for this commitment is below 50%.

---

## Components

|Component|Phase|Notes|Doc|
|---|---|---|---|
|Score Ring|1 only|Removed in Phase 2|`score_ring`|
|Commitment Info|1+|Unchanged across phases|`commitment_info`|
|Garment Display|2+|Replaces Score Ring|`garment_display`|
|Weekly Progress Table|2+|New in Phase 2|`commitment_weekly_progress` model|
|Micro-Insight|3|Pro / Premium only|`micro_insight`|

---

## Score Ring (Phase 1)

Displays today's `livePerformance` for this commitment as a circular arc. Driven by the pending instance for today, streamed live so it updates as the user logs. Replaced entirely by Garment Display in Phase 2.

---

## Garment Display (Phase 2)

The garment shows accumulated weaving progress over the commitment's lifetime — not today's progress. Two distinct values:

- `livePerformance` on today's instance — how much of today's target was achieved. Shown by Commitment Info.
- Garment completion percent — long-term accumulation of all weaving work since the commitment was created. Shown by Garment Display.

These are different numbers driven by different features. The garment completion percent comes from the Garment feature, not from the commitment or instance directly.

---

## Weekly Progress Table (Phase 2)

Ordered with the current week on top, oldest at the bottom. Shows the last 8 weeks, scrollable if more exist.

**Current week row** — live. Shows the running garment delta since Monday, updated after each day closes. Label: "This week (Mon–today)".

**Completed week rows** — sealed. Show the final garment delta for that Mon–Sun period.

**Negative delta rows** — labeled "decay". No bar, no color. Silence, not punishment. See `ux_principles` rule 7.

**Bar rendering** — relative widths. The largest positive delta in the visible rows gets full width. Others scale proportionally. Bars use the commitment's thread colors.

**First week state** — before any completed week exists, only the live row is shown with the message: "Your first week is still weaving."

---

## ⋮ Menu

Edit · Freeze · Mark as Complete · Delete

---

## + Log Button

Always visible at the bottom. Opens the full log sheet for this commitment. See `commitment_logging` component.

---

## Entry Points

No behavioral difference between entry points:

- Dashboard → tap commitment card
- Commitments screen → tap commitment row

---

## Navigation

- Back → returns to previous screen
- Edit (⋮ menu) → Commitment Form (edit mode)
- - Log → log sheet

---

## Data Sources

|Data|Source|Phase|
|---|---|---|
|Today's live performance|`CommitmentIdentityService.watchInstancesForDay(today)` — stream, read `livePerformance`|1+|
|Commitment definition|`CommitmentService.getDefinition(id)`|1+|
|Today's instance (progress, window)|`CommitmentIdentityService.watchInstancesForDay(today)` — stream|1+|
|Garment completion percent|GarmentService (Phase 2 feature) — stream|2+|
|Current week garment delta|GarmentService — stream|2+|
|Weekly history|GarmentService — one-time read|2+|
|AI insight|AIInsightService — one-time read|3|