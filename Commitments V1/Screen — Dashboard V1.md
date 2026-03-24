
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: commitment
**Phase**: 1
**Bottom nav:** tab 1 (leftmost)

**Purpose:** the user's daily home base. Open it to log commitments, check today's overall score, and track weekly momentum. Opened multiple times per day. Also the destination after day celebration.

---

## Layout

```
┌─────────────────────────────────────────┐
│  TODAY                                  │
│                                         │
│         [ Score Ring ]                  │
│           83%  → up to 92%              │
│                                         │
│  [ Performance Calendar — compact ]     │
│                                         │
│  [ Active ] [ Frozen ] [ Completed ]    │
│                                         │
│  ── DO ──────────────────────────────   │
│  [ Commitment Card ]                    │
│  [ Commitment Card ]                    │
│                                         │
│  ── AVOID ───────────────────────────   │
│  [ Commitment Card ]                    │
└─────────────────────────────────────────┘
```

---

## Components

|Component|Location|Value passed|Doc|
|---|---|---|---|
|Score Ring|Top center|`getDayScore(today)` + `getPossibleScore(today)`|`score_ring.md`|
|Performance Calendar (compact)|Below ring|`getWeeklyScores()`|`performance_calendar.md`|
|Commitment Card|DO and AVOID sections|Per-instance data|`commitment_card.md`|
|Commitment Logging|Gesture handler on each card|—|`commitment_logging.md`|
|Log Reward Animation|Fires on positive log|—|`log_reward_animation.md`|
|Context Tags|Sheet after certain logs|—|`context_tags.md`|
|Streaks|Milestone overlay on log|—|`streaks.md`|

**Phase 2 additions:**

- Thread Rope replaces ring inside each card

---

## Entry Points

**Normal navigation** — user taps Dashboard tab. Score ring sweeps to current score.

**After Day Celebration** — celebration screen auto-dismisses and navigates here. Score ring sweeps from zero — continuing the energy of the celebration.

**App open** — dashboard is the default screen. Score ring sweeps on enter.

---

## Filter Chips

`[ Active ] [ Frozen ] [ Completed ]` — Active default. Filters which cards are shown.

---

## Commitment Groups

Cards grouped by type — DO first, AVOID below. Tap card → Commitment Detail Screen.

---

## Navigation

- Tap commitment card → Commitment Detail Screen
- Bottom nav → Commitments Screen, Your Record, Reports
- `[+ Add]` → Commitment Form (via unlock engine gate)

---

## Data Sources

|Data|Method|Used by|
|---|---|---|
|Today's score|`CommitmentService.getDayScore(today)` — stream|Score Ring|
|Possible score|`CommitmentService.getPossibleScore(today)`|Score Ring|
|Weekly scores|`CommitmentService.getWeeklyScores()` — one-time read|Performance Calendar|
|Today's instances|`CommitmentService.getTodayInstances()` — stream|Commitment Cards|