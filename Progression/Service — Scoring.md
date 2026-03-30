**File Name**: service_scoring **Feature**: Progression (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** translates achievement subtypes into point values. Subscribes to `AchievementEarnedEvent`, looks up the point value for each subtype in `AppConfig`, and calls `ProgressionService.awardPoints()` directly. Internal to Progression — never called by features outside.

---

## Events Subscribed

### `AchievementEarnedEvent` → `_onAchievementEarned(event)`

```
points = _lookupPoints(event.record.subtype)
if points == 0: return   // unrecognised subtype — logged as warning
ProgressionService.awardPoints(points, event.record.subtype)
```

---

## Lookup Table

### `_lookupPoints(subtype)` → double

```
// cup achievements
bronze           → AppConfig.pointsBronzeCup
silver           → AppConfig.pointsSilverCup
gold             → AppConfig.pointsGoldCup
diamond          → AppConfig.pointsDiamondCup

// streak achievements
streakMilestone  → AppConfig.pointsStreakMilestone
globalBestStreak → AppConfig.pointsGlobalBestStreak

// garment achievements
garmentCompleted  → AppConfig.pointsGarmentCompleted
garmentFortified  → AppConfig.pointsGarmentFortified
garmentGolded     → AppConfig.pointsGarmentGolded
garmentDiamond    → AppConfig.pointsGarmentDiamond
```

Returns 0.0 for unrecognised subtypes — never throws. Logs a warning when 0 is returned so gaps are visible during development.

All point values are placeholders — tuned after launch from real user data. Only `AppConfig` needs to change.

---

## Rules

- Internal to Progression — never called by features outside
- All point values from `AppConfig` — never hardcoded
- Unrecognised subtypes return 0 and log a warning — never throw
- No state, no storage — pure translation layer

---

## Dependencies

- `AchievementService` — subscribes to `AchievementEarnedEvent`
- `ProgressionService.awardPoints()` — direct call
- `AppConfig` — all point values