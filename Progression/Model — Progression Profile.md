**File Name**: model_progression_profile **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** stores the user's cumulative progression state — total points earned, current level, and level achievement dates. One record per user. Created on first cup award and only ever updated — never recreated.

---

## Fields

```
ProgressionProfile
  totalCupPoints: double
  bonusPointsTotal: double
  currentLevel: int
  levelAchievedDates: List<DateTime?>
  lastBonusAwardedMonth: DateTime?
  updatedAt: DateTime
```

- **totalCupPoints** — cumulative cup points only. Never decreases. Level is calculated from this value alone — see `AppConfig.levelThresholds`.
- **bonusPointsTotal** — cumulative bonus thread points. Stored separately for transparency — never mixed into `totalCupPoints` and never used in level calculation. Display only.
- **currentLevel** — 0–7. Cached value derived from `totalCupPoints` via `getLevelForPoints()`. Cached to avoid recalculating on every read.
- **levelAchievedDates** — index matches level number. Index 0 = Apprentice start date. Index 1 = date Weaver was reached, and so on. Null entry means that level not yet reached. Append-only — dates written once when a level is first reached, never overwritten.
- **lastBonusAwardedMonth** — first day of the calendar month in which the last bonus was awarded. Enforces the one-bonus-per-month cap. Null means no bonus ever awarded. Comparison is month + year only, not exact datetime.
- **updatedAt** — updated on every save.

---

## Why Cup Points and Bonus Points Are Separate

`totalCupPoints` and `bonusPointsTotal` are stored separately and never summed for level calculation. Level is determined by `totalCupPoints` alone. Bonus points are shown separately in the UI for transparency — the user can always see exactly where their points came from. This prevents the bonus system from being gamed and keeps level progression honest and predictable.

If bonus points are later included in level calculation, this is a single-line change in `ProgressionService.getLevelForPoints()` — the model already supports both approaches.

---

## Rules

- One record per user — created on first cup award, never recreated
- `currentLevel` always derived from `totalCupPoints` using `AppConfig.levelThresholds`, then cached
- `levelAchievedDates` is append-only — never overwritten
- `bonusPointsTotal` is display-only — never used in level calculation
- Written only by `ProgressionService`
