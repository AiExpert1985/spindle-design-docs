**Created**: 17-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Progression **Phase**: 3

**Purpose:** manages the weaver progression system — points, levels, bonuses, and referral unlock. Purely reactive — subscribes to `CupEarnedEvent`. Never called directly by the scheduler.

---

## Events Subscribed

### `CupEarnedEvent`

→ `processCupEarned(event.cupLevel, event.weekStart, event.pointValue)`

Adds cup points to `totalCupPoints`. Checks for bonus. Recalculates level. If level changed → publishes `LevelReachedEvent`.

---

## Events Published

```
LevelReachedEvent
  newLevel: int
  levelName: String
  levelUpMessage: String     // verbatim copy from getLevelUpMessage() — subscribers never call ProgressionService for this
  previousLevel: int

BonusThreadAwardedEvent
  weekStart: DateTime
  bonusAmount: double
  trigger: String            // 'first_cup' | 'first_diamond' | 'three_consecutive'
                             // | 'comeback' | 'diamond_comeback'
                             // for AchievementsService narrative assembly
```

`BonusThreadAwardedEvent` is published inside `processCupEarned()` when a bonus is awarded. `AchievementsService` subscribes to write the bonus_thread achievement record.

---

## Level Table

|Level|Name|Cumulative points|
|---|---|---|
|0|Apprentice|0|
|1|Weaver|5|
|2|Journeyman|15|
|3|Artisan|25|
|4|Craftsman|40|
|5|Master Weaver|60|
|6|Loom Keeper|80|
|7|Penelope|105|

---

## Core Functions

### `processCupEarned(cupLevel, weekStart, pointValue)`

Steps:

1. Get current `ProgressionProfile` — create with defaults if first cup ever
2. Add `pointValue` to `totalCupPoints`
3. Call `checkAndAwardBonus(weekStart)` — if bonus awarded: a. Write `BonusThreadRecord` to `ProgressionRepository` b. Add to `bonusPointsTotal` c. Update `lastBonusAwardedMonth` d. Publish `BonusThreadAwardedEvent`
4. Recalculate `currentLevel` via `getLevelForPoints(totalCupPoints)`
5. If level changed → append date to `levelAchievedDates`, publish `LevelReachedEvent`
6. Save updated profile

The `BonusThreadRecord` is written before `BonusThreadAwardedEvent` is published — the record exists before any subscriber reacts. If the write fails, the event is not published.

Idempotent — checks if this week already processed.

---

### `checkAndAwardBonus(weekStart)`

Maximum one bonus per calendar month. Checks `lastBonusAwardedMonth` first.

Trigger evaluation (highest bonus first — only first match fires):

1. Diamond cup after a cupless week → +2.0
2. First diamond cup ever → +1.0
3. First cup ever earned → +1.0
4. Cup after 2+ cupless weeks (comeback) → +1.0
5. 3 cups in 3 consecutive weeks → +1.0

To evaluate triggers, reads cup history via `RewardService.fetchCupsSince(from)` — through the service interface, no repository exception.

Returns: `double` — bonus amount. 0.0 if no bonus.

---

### `getLevelForPoints(points)` — pure function

```
points < 5    → 0 (Apprentice)
points < 15   → 1 (Weaver)
points < 25   → 2 (Journeyman)
points < 40   → 3 (Artisan)
points < 60   → 4 (Craftsman)
points < 80   → 5 (Master Weaver)
points < 105  → 6 (Loom Keeper)
points >= 105 → 7 (Penelope)
```

### `getLevelName(level)` — pure function

Returns level name string. See level table above.

### `getLevelUpMessage(level)` — pure function

Returns verbatim notification copy per level. Used by `EncouragementService` and `NotificationService` via `LevelReachedEvent`.

```
1 → "You became a Weaver. First threads in place."
2 → "You reached Journeyman. The pattern is starting to show."
3 → "You reached Artisan. Consistent craft, recognizable work."
4 → "You reached Craftsman. Mastery is taking shape."
5 → "You reached Master Weaver. Rare discipline."
6 → "You reached Loom Keeper. You tend the craft itself now."
7 → "You reached Penelope. Twenty years of discipline — thread by thread."
```

### `getProgressionSummary()` — one-time read

Returns full progression state for the Progression Screen.

### `watchProgressionSummary()` — stream

Used by Your Record screen teaser.

### `getCurrentWeekProjection()` — one-time read

Returns projected cup and points based on current week's performance ratio from `PerformanceAccountingService.getLiveWeekRatio()`.

### `isReferralUnlocked(tier)` — pure function

Returns true if `currentLevel >= 2` and tier is Pro or Premium.

### `backfillMissingProgression()`

Runs on startup. Finds weeks with cups but no progression record. Calls `processCupEarned()` for each silently. No event published for backfilled progression.

---

## Rules

- Subscribes to `CupEarnedEvent` — never called directly by scheduler
- Reads cup history through `RewardService` — no repository exception anywhere
- `getLevelForPoints()`, `getLevelName()`, `getLevelUpMessage()` are pure functions
- `bonusPointsTotal` is display-only — never used in level calculation
- Publishes `LevelReachedEvent` — Encouragement and Notification subscribe independently

---

## Rules

- `BonusThreadRecord` is written before `BonusThreadAwardedEvent` is published — record exists before any subscriber reacts
- Bonus records are permanent — never deleted, never updated
- `checkAndAwardBonus()` reads bonus history from `ProgressionRepository.fetchBonusSince()` for trigger evaluation — replaces the previous pattern of reading `RewardRepository` directly

---

## Dependencies

- `EventBus` — subscribes to events, publishes events
- `RewardService` — reads cup history for bonus trigger evaluation
- `PerformanceAccountingService` — reads live week ratio for projection
- `ProgressionRepository` — reads/writes ProgressionProfile and BonusThreadRecord history