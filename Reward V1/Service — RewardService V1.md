**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Rewards **Phase**: 1 (streaks) · Phase 2 (cups)

**Purpose:** calculates and stores earned rewards — weekly cups and streak milestones. Purely reactive — subscribes to domain events. Never called directly by the scheduler or any other feature.

---

## Events Subscribed

### `ActivityRecordedEvent`

→ `processStreak(event.definitionId, event.logResult)`

Reads definition, calculates new streak via `updateStreak()` pure function, saves via `CommitmentService.saveDefinitionState()`. If milestone reached → writes `StreakMilestoneRecord` to `RewardRepository` → publishes `StreakMilestoneReachedEvent`.

The record is written before the event is published. Both happen atomically in the sense that if the write fails, the event is not published — the milestone is not announced until it is safely stored.

### `WeeklyPerformanceCalculatedEvent`

→ `processWeeklyCup(event.weekStart, event.ratio)`

Receives pre-calculated ratio from Performance. Checks idempotency. Determines cup level. Saves record. Publishes `CupEarnedEvent` if cup earned.

---

## Events Published

```
CupEarnedEvent
  cupLevel: bronze | silver | gold | diamond
  weekStart: DateTime
  pointValue: double

StreakMilestoneReachedEvent
  definitionId: String
  commitmentName: String     // for AchievementsService — avoids service call at write time
  streakCount: int
  badge: String
```

---

## Cup Thresholds

|Cup|Weekly performance ratio|Point value|
|---|---|---|
|🥉 Bronze|≥ 0.60|1|
|🥈 Silver|≥ 0.75|2|
|🥇 Gold|≥ 0.85|3|
|💎 Diamond|≥ 0.95|5|

Below 0.60 — no cup, no record, no event.

---

## Cup Functions

### `processWeeklyCup(weekStart, ratio)`

Idempotent — exits if record already exists for this `weekStart`.

### `backfillMissingCups()`

Runs on app startup. For each past week with no cup record, fetches ratio from `PerformanceAccountingService.getWeeklyRatio(weekStart)` and calls `processWeeklyCup()`. No event published for backfilled cups.

### `fetchAllCups()`

Returns all cup records ordered by date. Used by: weekly cups component, progression screen popup.

### `fetchCupsSince(from)`

Returns cup records since a date. Used by: `ProgressionService` bonus trigger evaluation.

---

## Streak Functions

### `updateStreak(definition, logResult)` — pure function

- Kept → increment `currentStreak`
- Missed → reset `currentStreak` to 0
- `currentStreak` > `bestStreak` → update `bestStreak`
- Milestone hit → return `milestoneReached: true`

Returns: `{ updatedDefinition, milestoneReached: bool }`

### `getStreakMilestone(streak)` — pure function

|Days|Badge|
|---|---|
|3|🥉 Bronze|
|5|🥈 Silver|
|7|🥇 Gold|
|10|🏆 Trophy|
|14|💎 Diamond|

---

## Rules

- Subscribes to domain events — never called directly by any feature or scheduler
- `updateStreak()` is pure — never saves, never calls any service
- Cup thresholds are ratios not raw percentages
- `StreakMilestoneRecord` is written before `StreakMilestoneReachedEvent` is published — record exists before any subscriber reacts
- Milestone records are permanent — never deleted, never updated
- Publishes events after every successful state change

---

## Dependencies

- `EventBus` — subscribes to events, publishes events
- `CommitmentService` — reads definitions, saves streak state
- `PerformanceAccountingService` — reads weekly ratio for backfill only
- `RewardRepository` — cup record storage and streak milestone record storage