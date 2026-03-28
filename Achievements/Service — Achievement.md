**File Name**: service_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the single entry point for recording achievements. Producing features call `addAchievement()` directly — no event subscriptions needed. Stores `AchievementRecord`, publishes `AchievementEarnedEvent`, and serves all achievement reads from its own collection.

---

## Design Decisions

**Why producing features call `addAchievement()` directly instead of publishing events.** The original design had Achievements subscribing to events from multiple producing features. This meant Achievements had to sit above all of them, but Progression needed to sit above Achievements — creating a long subscription chain where adding a new producing feature required modifying `AchievementService`. Moving Achievements below all producing features inverts this: producing features call downward into a stable API. Adding a new producing feature requires zero changes here.

**Why achievement detection is internal to each producing service.** Each service already knows exactly when its achievement moment happens and already has all the data needed. A separate Milestones feature existed only to watch `StreakChangedEvent` and translate threshold crossings — that was a middleman with no added value. `StreakService` detects milestones directly inside `_detectAchievements()`. `GarmentService` detects completions inside `_detectAchievements()`. The logic is a few lines in the right place — no separate feature needed.

**Why `AchievementService` is the single source of truth.** Every achievement flows through `addAchievement()`. All queries — cups history, streak records, best streak, garment completions — are answered from the `AchievementRecord` collection. No producing feature needs to be called for achievement data. Callers get one consistent interface regardless of which feature produced the achievement.

**Why records are never deleted.** Achievements are facts about the user's history. Deleting a commitment does not erase what the user accomplished with it. The `sourceId` may become stale but the achievement remains valid for display and scoring.

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

- `CupService._addAchievement()` — when a cup is earned
- `StreakService._addAchievement()` — when a streak milestone or global best is crossed
- `GarmentService._addAchievement()` — when a garment completes

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

Live stream. Used by the achievements screen during a session.

### `getCupHistory(from, to)` → List<AchievementRecord>

Returns achievements where `type: cup` in the period.

### `getCupsSince(from)` → List<AchievementRecord>

Returns achievements where `type: cup` and `createdAt >= from`.

### `getBestStreak(definitionId?)` → AchievementRecord?

Returns the most recent achievement where `subtype: globalBestStreak`. If `definitionId` provided, filters to that commitment. Otherwise returns the all-time global best.

### `getStreakAchievements(definitionId, from, to)` → List<AchievementRecord>

Returns achievements where `type: streak` for a specific commitment in the period. Used by commitment detail screen and achievement detail sheet.

---

## Rules

- `addAchievement()` is the only write path — no other service writes `AchievementRecord`
- `AchievementEarnedEvent` published only after successful write
- No event subscriptions — all producing features call directly
- All reads are self-contained — no calls to Cups, Streak, Garment, or any other feature
- Records are permanent — never deleted for any reason
- Producing features guarantee no duplicate calls — `existsForSource` is the safety net
- All reads use a time window

---

## Dependencies

- `AchievementRepository` — reads and writes `AchievementRecord`
- Publishes `AchievementEarnedEvent` on its own stream

---

## Later Improvements

**Rewards feature.** A `RewardService` that evaluates sustained rolling performance (e.g. 30 days above threshold) and calls `addAchievement()` with `type: reward, subtype: periodicReward`. When added, it sits above Achievements in the chain and calls downward — zero changes needed here. The `periodicReward` enum value and `AppConfig` point entry are already present.