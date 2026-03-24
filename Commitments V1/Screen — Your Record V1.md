
**Created**: 15-Mar-2026 
**Modified**: 17-Mar-2026 
**Feature**: Commitment 
**Phase**: 1 (sections 1, 2 partial, 3) · Phase 2 (weekly cups, full mode calendar, personal bests) · Phase 3 (progression teaser) 

**Access:** Threads screen slots teaser · deep-link from notifications (cup earned, streak milestone, slot unlocked)

**Not in bottom nav.**

**Purpose:** the user's complete progress picture in one scroll. Shows commitment health, consistency history, earned rewards, streak records, and (Phase 3) a teaser for the progression journey. Reflective screen — opened a few times a week, not daily.

---

## Layout

```
┌─────────────────────────────────────────┐
│  YOUR RECORD                            │
│  Consistency: 76%  ·  Best streak: 14d  │
│                                         │
│  Artisan  ●●●  32 pts  →ˢ              │  ← Phase 3 only
│                                         │
│  🥉   🥈   🥇   💎                      │
│   8    5    3    1                      │
├─────────────────────────────────────────┤
│  SECTION 1 — YOUR COMMITMENTS           │
│                                         │
│  [ Commitment Health Row ]              │
│  5 active  ·  2 slots available         │
│  Free tier: 5 of 7  [ Upgrade to Pro ] │
├─────────────────────────────────────────┤
│  SECTION 2 — YOUR HISTORY               │
│                                         │
│  [ Performance Calendar — full mode ]   │
│       ← November →                      │
│  Mon  Tue  Wed  Thu  Fri  Sat  Sun      │
│   ◉    ⬤   ◉    ·    ●    ⬤   ·       │
│                                         │
├─────────────────────────────────────────┤
│  SECTION 3 — STREAKS                    │
│                                         │
│  [ Streaks Component ]                  │
│  Morning Walk   7 days 🥇  Best: 12d   │
│  Read Daily     3 days 🥉  Best: 21d   │
└─────────────────────────────────────────┘
```

---

## Progression Teaser (Phase 3)

Shown in the header, between summary numbers and the cups row. Compact — one line only.

```
Artisan  ●●●  32 pts  →
```

- Level name, compact progress dots, total points, arrow
- Tap anywhere on this row → navigates to `/progression`
- Hidden entirely in Phase 1 and Phase 2 — not shown as disabled, just absent
- Updates in real time via `ProgressionService.watchProgressionSummary()` stream

---

## Components

|Component|Location|Doc|
|---|---|---|
|Progression Teaser|Header — between summary and cups|Phase 3 — see `screen_progression.md`|
|Weekly Cups|Header — below progression teaser|`weekly_cups.md`|
|Commitment Health Row|Section 1|`commitment_health_row.md`|
|Unlock Engine|Section 1 — slot count and gate status|`unlock_engine.md`|
|Performance Calendar (full mode)|Section 2|`performance_calendar.md`|
|Streaks|Section 3|`streaks.md`|

---

## Header

Two summary numbers across all active commitments, last 30 days:

- **Consistency rate** — percentage of days with any performance dot
- **Best streak** — longest streak across all commitments

---

## Navigation

Deep-link: `/record` · `/record?section=cups` · `/record?section=streaks`

Tap progression teaser → `/progression`

---

## Data Sources

|Data|Source|
|---|---|
|Consistency rate, best streak|Computed from `CommitmentService.getPeriodAverageScore()`|
|Progression teaser|`ProgressionService.watchProgressionSummary()` — stream (Phase 3)|
|Weekly cups|`RewardService`|
|Commitment health dots|`CommitmentService.getTodayInstances()`|
|Monthly history|`CommitmentService.getMonthScores(month, year)`|
|Streaks|Thread definitions via `CommitmentService.getActiveCommitments()`|