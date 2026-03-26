**File Name**: service_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** pure aggregator. Subscribes to events from all achievement-producing features (Cups, Rewards, Milestones), calls `toAchievementRecord()` on each event's model via the `Achievable` interface, writes the unified `AchievementRecord`, and publishes `AchievementEarnedEvent`. The single public read interface for achievement data.

---

## Design Decisions

**Why AchievementService is a pure aggregator.** Each achievement-producing feature (Cups, Rewards, Milestones) is independent — it owns its own model, repository, and logic. Achievements aggregates their outputs without knowing their internals. Adding a new achievement-producing feature requires no change to `AchievementService` — only a new feature that publishes an event carrying an `Achievable` model.

**Why `toAchievementRecord()` is called here.** The producing feature publishes its domain event. `AchievementService` decides when and how to translate that into an `AchievementRecord`. The translation is a one-liner because `Achievable` enforces the data contract — but the decision to translate lives here.

**Why Milestones is no longer internal.** Milestones has its own model, repository, and service — it qualifies as an independent feature by the same criteria as Cups and Rewards. Keeping it internal would make Achievements a mixed-concern feature again. `AchievementService` now subscribes to `MilestoneEarnedEvent` from Milestones exactly as it subscribes to `CupEarnedEvent` and `RewardEarnedEvent`.

---

## Events Subscribed

### `CupEarnedEvent` → `_onCupEarned(event)`

```
record = event.cup.toAchievementRecord()
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `RewardEarnedEvent` → `_onRewardEarned(event)`

```
record = event.record.toAchievementRecord()
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `MilestoneEarnedEvent` → `_onMilestoneEarned(event)`

```
record = event.record.toAchievementRecord()
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Cleans up `AchievementRecord` entries linked to this `definitionId`.

---

## Events Published (public)

```
AchievementEarnedEvent
  record: AchievementRecord
```

Consumed by `ScoringService` (Progression) and `EncouragementService`.

---

## Public Read Functions

### `getAchievements(limit?, type?)` → List<AchievementRecord>

Ordered by earnedAt descending. Used by the achievements screen.

### `watchRecentAchievements(limit?)` → Stream<List<AchievementRecord>>

Live stream. Used by the achievements screen.

### `getCupHistory()` → List<WeeklyCup>

Proxied from `CupService`. Used by the Progression screen.

### `getCupsSince(from)` → List<WeeklyCup>

Proxied from `CupService`. Used by `ScoringService` for bonus trigger evaluation.

### `getStreakRecord(definitionId)` → StreakRecord?

Proxied from `StreakService`. Used by the commitment detail screen and Your Record screen.

### `getBestStreakOverall()` → int

Proxied from `StreakService`. Used by the Your Record screen.

---

## Rules

- Calls `toAchievementRecord()` on every received `Achievable` model — never constructs `AchievementRecord` manually
- `AchievementEarnedEvent` published only after the record is successfully written
- Never imports internal logic from Cups, Rewards, Milestones, or Streak — only their public events and the `Achievable` interface
- Idempotency delegated to producing features — each prevents duplicate records before publishing

---

## Dependencies

- EventBus — subscribes to `CupEarnedEvent` (Cups), `RewardEarnedEvent` (Rewards), `MilestoneEarnedEvent` (Milestones), `InstancePermanentlyDeletedEvent` (Commitment); publishes `AchievementEarnedEvent`
- `AchievementRepository` — writes and reads `AchievementRecord`
- `CupService` — proxied reads
- `StreakService` — proxied reads