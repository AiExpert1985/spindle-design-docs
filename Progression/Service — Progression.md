**File Name**: service_progression **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** manages the weaver progression system — points, levels, and bonus threads. Purely reactive — subscribes to `CupEarnedEvent` only. Never called directly by a scheduler.

---

## Events Subscribed

### `CupEarnedEvent` → `processCupEarned(event)`

Adds cup points, checks for bonus, recalculates level. If level changed → publishes `LevelReachedEvent`.

---

## Events Published

```
LevelReachedEvent
  newLevel: int
  levelName: String
  levelUpMessage: String
  previousLevel: int
```

```
BonusThreadAwardedEvent
  weekStart: DateTime
  bonusAmount: double
  trigger: BonusTrigger
```

`levelUpMessage` is carried in the event so subscribers never need to call back into `ProgressionService` to get the copy.

`BonusThreadAwardedEvent` is published inside `processCupEarned()` when a bonus is awarded. `AchievementsService` subscribes to write the achievement record.

---

## Core Function

### `processCupEarned(event)`

Steps in order:

1. Get current `ProgressionProfile` — create with defaults if first cup ever
2. Add `event.pointValue` to `totalCupPoints`
3. Call `checkAndAwardBonus(event.weekStart)` — if bonus awarded:
   - Write `BonusThreadRecord` to `ProgressionRepository`
   - Add to `bonusPointsTotal`
   - Update `lastBonusAwardedMonth`
   - Publish `BonusThreadAwardedEvent`
4. Recalculate `currentLevel` via `getLevelForPoints(totalCupPoints)`
5. If level changed → append date to `levelAchievedDates`, publish `LevelReachedEvent`
6. Save updated profile

`BonusThreadRecord` is written before `BonusThreadAwardedEvent` is published — the record exists before any subscriber reacts. If the write fails, the event is not published.

Idempotent — checks if this week's cup has already been processed before running.

---

## Bonus Evaluation

### `checkAndAwardBonus(weekStart)` → double

Maximum one bonus per calendar month. Checks `lastBonusAwardedMonth` first — exits immediately if a bonus was already awarded this month.

Trigger evaluation — highest bonus first, only first match fires:

| Priority | Trigger | Bonus |
|---|---|---|
| 1 | Diamond cup after a cupless week | +2.0 |
| 2 | First diamond cup ever | +1.0 |
| 3 | First cup ever earned | +1.0 |
| 4 | Cup after 2+ cupless weeks (comeback) | +1.0 |
| 5 | 3 cups in 3 consecutive weeks | +1.0 |

Reads cup history via `RewardService` — never accesses `RewardRepository` directly.

Returns: bonus amount as double. 0.0 if no bonus awarded.

---

## Pure Functions

### `getLevelForPoints(points)` → int

Returns level 0–7 based on `AppConfig.levelThresholds`. See `AppConfig` for threshold values.

### `getLevelName(level)` → String

Returns level name. See `language_guide` for the full level name table.

### `getLevelUpMessage(level)` → String

Returns verbatim notification copy per level. Fixed strings — never improvised.

```
1 → "You became a Weaver. First threads in place."
2 → "You reached Journeyman. The pattern is starting to show."
3 → "You reached Artisan. Consistent craft, recognizable work."
4 → "You reached Craftsman. Mastery is taking shape."
5 → "You reached Master Weaver. Rare discipline."
6 → "You reached Loom Keeper. You tend the craft itself now."
7 → "You reached Penelope. Twenty years of discipline — thread by thread."
```

### `isReferralUnlocked(tier)` → bool

Returns true if `currentLevel >= 2` and tier is Pro or Premium.

---

## Read Functions

### `getProgressionSummary()` — one-time read

Returns full progression state for the Progression screen.

### `watchProgressionSummary()` — stream

Used by the Your Record screen teaser.

### `getCurrentWeekProjection()` — one-time read

Returns projected cup and points based on the current week's live performance score. Reads from `PerformanceService.getOverallWeekScore(currentWeekStart)` and maps the ratio to the nearest cup threshold.

---

## Startup Function

### `backfillMissingProgression()`

Runs on app startup. Finds weeks with cups but no corresponding progression record — can occur after reinstall or data migration. Calls `processCupEarned()` for each silently. No event published for backfilled records.

---

## Rules

- Subscribes to `CupEarnedEvent` only — never called directly by a scheduler
- Reads cup history through `RewardService` — never accesses `RewardRepository` directly
- `bonusPointsTotal` is display-only — never used in level calculation
- `getLevelForPoints()`, `getLevelName()`, `getLevelUpMessage()`, `isReferralUnlocked()` are pure functions
- `BonusThreadRecord` written before `BonusThreadAwardedEvent` published — record always exists before subscribers react

---

## Dependencies

- EventBus — subscribes to `CupEarnedEvent`; publishes `LevelReachedEvent`, `BonusThreadAwardedEvent`
- `RewardService` — reads cup history for bonus trigger evaluation
- `PerformanceService` — reads current week score for projection
- `ProgressionRepository` — reads and writes `ProgressionProfile` and `BonusThreadRecord`
- `AppConfig` — level thresholds, bonus amounts
- `TemporalHelper` — current week start calculation
