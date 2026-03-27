**File Name**: service_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** pure aggregator. Subscribes to events from all achievement-producing features (Cups, Rewards, Milestones), calls `toAchievementRecord()` on each event's model via the `Achievable` interface, writes the unified `AchievementRecord`, and publishes `AchievementEarnedEvent`. The single public read interface for achievement data.

---

## Design Decisions

**Why AchievementService is a pure aggregator.** Each achievement-producing feature (Cups, Rewards, Milestones) is independent — it owns its own model, repository, and logic. Achievements aggregates their outputs without knowing their internals. Adding a new achievement-producing feature requires no change to `AchievementService` — only a new feature that publishes an event carrying an `Achievable` model.

**Why `toAchievementRecord()` is called here.** The producing feature publishes its domain event. `AchievementService` decides when and how to translate that into an `AchievementRecord`. The translation is a one-liner because `Achievable` enforces the data contract — but the decision to translate lives here.

**Why Milestones is independent, not internal.** Milestones has its own model, repository, and service — it qualifies as an independent feature by the same criteria as Cups and Rewards. `AchievementService` subscribes to `MilestoneEarnedEvent` from Milestones exactly as it subscribes to `CupEarnedEvent` and `RewardEarnedEvent`.

**Why achievement records are never deleted.** Achievements represent moments in the user's history — earning a diamond cup, reaching a 7-day streak. These are permanent facts. Deleting a commitment does not erase what the user achieved with it. The `sourceId` on a record may become stale, but the achievement itself remains.

---

## Events Subscribed

### `CupService.onCupEarned` → `_onCupEarned(event)`

```
if AchievementRepository.existsForSource(event.cup.id): return  // safety net
record = event.cup.toAchievementRecord()
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `RewardService.onRewardEarned` → `_onRewardEarned(event)`

```
if AchievementRepository.existsForSource(event.record.id): return
record = event.record.toAchievementRecord()
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `MilestoneService.onMilestoneEarned` → `_onMilestoneEarned(event)`

```
if AchievementRepository.existsForSource(event.record.id): return
record = event.record.toAchievementRecord()
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

---

## Events Published

```
AchievementEarnedEvent
  record: AchievementRecord
```

Published only after the record is successfully written. Consumed by `ScoringService` (Progression) and `EncouragementService`.

---

## Public Read Functions

### `getAchievements(from, to, type?) → List<AchievementRecord>`

All achievement records in the date range, optionally filtered by type. Ordered by `createdAt` descending. Used by the achievements screen.

### `watchRecentAchievements(from, to) → Stream<List<AchievementRecord>>`

Live stream. Used by the achievements screen for live updates during a session.

### `getCupHistory(from, to) → List<WeeklyCup>`

Proxied from `CupService.getAllCups(from, to)`. Used by the Progression screen and achievement detail sheet.

### `getCupsSince(from) → List<WeeklyCup>`

Proxied from `CupService.getCupsSince(from)`. Used by `ScoringService` for bonus trigger evaluation.

### `getStreakRecord(definitionId) → StreakRecord?`

Proxied from `StreakService`. Used by the commitment detail screen and achievement detail sheet.

### `getBestStreakOverall() → GlobalBestStreakRecord?`

Proxied from `StreakService`. Returns the full `GlobalBestStreakRecord` — which commitment, how many days, when achieved. Used by the Your Record screen.

---

## Rules

- Calls `toAchievementRecord()` on every received `Achievable` model — never constructs `AchievementRecord` manually
- `AchievementEarnedEvent` published only after the record is successfully written
- Never imports internal logic from Cups, Rewards, Milestones, or Streak — only their public events and the `Achievable` interface
- Producing features guarantee no duplicate events through their own idempotency — `existsForSource` is a secondary safety net
- Achievement records are permanent — never deleted based on commitment deletion or any other event
- All read queries use a time window — never fetch unbounded history

---

## Dependencies

- `CupService` — subscribes to `onCupEarned`; proxied reads `getCupHistory()`, `getCupsSince()`
- `RewardService` — subscribes to `onRewardEarned`
- `MilestoneService` — subscribes to `onMilestoneEarned`
- `StreakService` — proxied reads `getStreakRecord()`, `getBestStreakOverall()`
- `AchievementRepository` — writes and reads `AchievementRecord`