
**Created**: 17-Mar-2026 
**Modified**: - 
**Feature**: Progression 
**Phase**: 3

**Purpose:** stores the user's cumulative progression state — total points earned, current level, bonus thread history, and level achievement dates. One record per user, created on first cup award and only ever updated — never recreated.

---

## Fields

```
totalCupPoints: double              // cumulative cup points only — never decreases
bonusPointsTotal: double            // cumulative bonus thread points — shown separately in UI
                                    // for transparency, never mixed into totalCupPoints
currentLevel: int                   // 0–7, cached and updated on every point change
                                    // derived from totalCupPoints via getLevelForPoints()
                                    // cached here to avoid recalculating on every read
levelAchievedDates: List<DateTime>  // index matches level number (index 0 = Apprentice start)
                                    // index 1 = date Weaver was reached, etc.
                                    // null entry means that level not yet reached
lastBonusAwardedMonth: DateTime?    // first day of the calendar month in which the last
                                    // bonus was awarded — enforces once-per-month cap
                                    // null means no bonus ever awarded

updatedAt: DateTime
```

---

## Level Table (reference — lives here as the single source of truth)

|Level|Name|Points to reach this level|Cumulative points required|
|---|---|---|---|
|0|Apprentice|—|0|
|1|Weaver|5|5|
|2|Journeyman|10|15|
|3|Artisan|10|25|
|4|Craftsman|15|40|
|5|Master Weaver|20|60|
|6|Loom Keeper|20|80|
|7|Penelope|25|105|

Points accumulate permanently. Levels are never lost. A bad week adds 0 points — it never subtracts.

---

## Cup Point Values (reference)

|Cup|Weekly avg score|Point value|
|---|---|---|
|🥉 Bronze|60–74%|1|
|🥈 Silver|75–84%|2|
|🥇 Gold|85–94%|3|
|💎 Diamond|≥ 95%|5|

---

## Bonus Thread Triggers (reference)

Maximum one bonus award per calendar month. Checked by comparing `lastBonusAwardedMonth` to the current month before any bonus is written.

|Trigger|Bonus|Repeatable|
|---|---|---|
|First cup ever earned|+1|Once lifetime|
|First diamond cup ever|+1|Once lifetime|
|3 cups in 3 consecutive weeks|+1|Yes, monthly cap applies|
|Cup earned after 2+ cupless weeks (comeback)|+1|Yes, monthly cap applies|
|Diamond cup earned after a cupless week|+2|Yes, monthly cap applies|

---

## Important Separation

`totalCupPoints` and `bonusPointsTotal` are stored separately and never summed for level calculation. **Level is determined by `totalCupPoints` alone.** Bonus points are displayed for transparency but do not affect level thresholds. This prevents the bonus system from being gamed and keeps the level calculation honest and predictable.

If you later decide to include bonuses in level calculation, this is a single-line change in `ProgressionService.getLevelForPoints()` — the model already supports both approaches.

---

## Rules

- One record per user — created on first cup award, never recreated
- `currentLevel` is always derived from `totalCupPoints` using the level table above, then cached here
- `levelAchievedDates` is append-only — dates are written once when a level is first reached, never overwritten
- `lastBonusAwardedMonth` stores the first day of the month (00:00:00) — comparison is month + year only, not exact datetime
- `bonusPointsTotal` is display-only — never used in level calculations
- `updatedAt` is updated on every save