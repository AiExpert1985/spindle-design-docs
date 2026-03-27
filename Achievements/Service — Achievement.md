**File Name**: service_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the single entry point for recording achievements. Producing features call `addAchievement()` directly — no event subscriptions needed. Stores `AchievementRecord`, publishes `AchievementEarnedEvent`, and serves all achievement reads from its own collection.

---

## Design Decisions

**Why producing features call `addAchievement()` directly instead of publishing events.** The original design had Achievements subscribing to events from Cups, Rewards, Milestones, Streak, and Garment. This meant Achievements sat above all those features — but Progression also needed to sit above Achievements. This created a long subscription chain where adding a new producing feature required modifying `AchievementService`. Moving Achievements below all producing features inverts this: producing features call downward into a stable API. Adding a new producing feature requires zero changes to `AchievementService`.

**Why `AchievementService` is the single source of truth for all achievement data.** Every achievement flows through `addAchievement()`. This means the `AchievementRecord` collection contains everything — cups history, streak milestones, garment completions, rewards. All queries are answered from this one collection. No feature needs to be called for achievement data. Callers get one consistent interface regardless of which feature produced the achievement.

**Why reads like `getCupHistory` and `getBestStreak` live here.** These are filtered reads on the `AchievementRecord` collection — `type: cup` for cups, `subtype: globalBestStreak` for best streak. No external service calls needed. `AchievementService` owns the data and serves it directly. This keeps Progression, Encouragement, and screens from needing to know which feature produced which achievement.

---

## Public Write Function

### `addAchievement(AchievementRecord record)`

The single entry point for all achievement recording.

```
if AchievementRepository.existsForSource(record.sourceId): return  // safety net
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

Called by producing features at the achievement moment:

- `CupService` — when a cup is earned
- `RewardService` — when a reward is earned
- `MilestoneService` — when a streak milestone is crossed
- `StreakService` — when a new global best streak crosses a milestone
- `GarmentService` — when a garment reaches completion

---

## Events Published

```
AchievementEarnedEvent
  record: AchievementRecord
```

Published after every successful write. Consumed by `ScoringService` (Progression) and `EncouragementService`.

---

## Public Read Functions

All reads query the `AchievementRecord` collection internally — no calls to other features.

### `getAchievements(from, to, type?, subtype?, definitionId?)` → List<AchievementRecord>

Unified read. All filters optional. Ordered by `createdAt` descending.

### Convenience wrappers

```dart
getAchievementsForWeek(DateTime weekStart)
getAchievementsForMonth(DateTime month)
getAchievementsForPeriod(DateTime from, DateTime to)
```

### `watchAchievements(from, to)` → Stream<List<AchievementRecord>>

Live stream. Used by the achievements screen.

### `getCupHistory(from, to)` → List<AchievementRecord>

Returns achievements where `type: cup` in the period. Ordered by `createdAt` descending.

### `getCupsSince(from)` → List<AchievementRecord>

Returns achievements where `type: cup` and `createdAt >= from`.

### `getBestStreak(definitionId?)` → AchievementRecord?

Returns the achievement where `subtype: globalBestStreak`. If `definitionId` provided, filters to that commitment. Otherwise returns the all-time global best.

### `getStreakAchievements(definitionId, from, to)` → List<AchievementRecord>

Returns achievements where `type: streak` or `type: milestone` for a specific commitment. Used by commitment detail screen.

---

## Rules

- `addAchievement()` is the only write path — no other service writes `AchievementRecord`
- `AchievementEarnedEvent` published only after successful write
- All reads are self-contained — no calls to Cups, Streak, Milestones, Garment, or Rewards
- Records are permanent — never deleted for any reason
- Idempotency: producing features prevent duplicate calls; `existsForSource` is the safety net
- All reads use a time window

---

## Dependencies

- `AchievementRepository` — reads and writes `AchievementRecord`
- Publishes `AchievementEarnedEvent` on its own stream