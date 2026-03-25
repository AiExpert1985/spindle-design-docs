**File Name**: model_progression_profile **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** stores the user's cumulative progression state. One record per user. Created on first points awarded and only ever updated — never recreated.

---

## Fields

```
ProgressionProfile
  totalPoints: double
  bonusPointsTotal: double
  currentLevel: int
  levelAchievedDates: List<DateTime?>
  updatedAt: DateTime
```

- **totalPoints** — all accumulated points from all achievement types. Never decreases. Level is derived from this value alone via `AppConfig.levelThresholds`.
- **bonusPointsTotal** — running total of bonus points only. Display cache — stored so the Progression screen can show "32 pts total · 3 from bonuses" without a cross-feature read. Never used in level calculation.
- **currentLevel** — 0–7. Cached value derived from `totalPoints`. Cached to avoid recalculating on every read.
- **levelAchievedDates** — index matches level number. Index 0 = Apprentice start date. Null entry means that level not yet reached. Append-only — dates written once when a level is first reached, never overwritten.
- **updatedAt** — updated on every save.

---

## Rules

- One record per user — created on first `PointsAwardedEvent`, never recreated
- `currentLevel` always derived from `totalPoints` using `AppConfig.levelThresholds`, then cached
- `levelAchievedDates` is append-only — never overwritten
- `bonusPointsTotal` is a display cache — updated when `isBonus: true` on `PointsAwardedEvent`
- Written only by `ProgressionService`