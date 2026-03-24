**Created**: 18-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Performance **Phase**: 2

**Purpose:** converts closed commitment windows into CommitmentDailyEntry records. Subscribes to window domain events published by `CommitmentSchedulerService` — never to the scheduler tick directly. Single source of truth for all numerical performance data.

---

## Events Subscribed

### `WindowClosedEvent`

→ `processWindowClose(event.instanceId, event.definitionId, event.windowStart, event.windowEnd)`

Primary trigger for performance calculation. One entry written per window close. Publishes `PerformanceEntryWrittenEvent` after writing.

### `WeekEndedEvent`

→ `processWeekEnd(event.weekStart, event.weekEnd)`

Calculates weekly performance ratio across all active commitments for the week just ended. Publishes `WeeklyPerformanceCalculatedEvent(weekStart, ratio)`.

### `WeekStartedEvent`

→ `createLiveWeekRecords(event.weekStart)`

Creates live `CommitmentWeeklyProgress` records for all active commitments for the new week.

### `CommitmentFrozenEvent`

→ reserved for future cleanup logic — no current action needed since frozen commitments produce no `WindowClosedEvent`

### `CommitmentPermanentlyDeletedEvent`

→ `deleteEntriesForCommitment(event.definitionId)`

Performance owns its own data deletion.

---

## Events Published

```
PerformanceEntryWrittenEvent
  definitionId: String
  netChange: double
  date: DateTime

WeeklyPerformanceCalculatedEvent
  weekStart: DateTime
  ratio: double
```

---

## Configuration Constants

All constants here — single place to change.

```dart
static const Map<RecurrenceType, int> targetRepetitions = {
  RecurrenceType.daily:   30,
  RecurrenceType.weekly:  20,
  RecurrenceType.monthly: 12,
  RecurrenceType.custom:  20,
};

static const int decayThreshold = 2;
static const double decayAmountPerWindow = 1.0;
static const int streakBonusThreshold = 3;
static const double streakBonusPerWindow = 1.0;

static const Map<double, double> performanceBonusTiers = {
  0.10: 1.0,
  0.30: 2.0,
  0.60: 3.0,
};

static double getDailyTarget(RecurrenceType type) =>
    100.0 / targetRepetitions[type]!;
```

---

## Core Function

### `processWindowClose(instanceId, definitionId, windowStart, windowEnd)`

Idempotent — checks if entry already exists for this `instanceId` before running.

```
1. Read closed instance result via ActivityService.getInstanceResult()
2. Read definition via CommitmentService.getDefinition()
3. If isFrozen or isCompleted → exit
4. Calculate baseContribution, decayPenalty, streakBonus, performanceBonus
5. Apply avoid direction if needed
6. Clamp netChange so garment stays 0.0–100.0
7. Write CommitmentDailyEntry
8. Publish PerformanceEntryWrittenEvent
```

---

## Weekly Performance Ratio

### `processWeekEnd(weekStart, weekEnd)`

```
for each active non-frozen commitment with windows closed this week:
  commitmentRatio = sum of netChange ÷ (getDailyTarget(recurrenceType) × windowsClosed)

weeklyRatio = average of all commitmentRatios
```

Publishes `WeeklyPerformanceCalculatedEvent`. Properties: fair regardless of commitment count, fair regardless of recurrence type, frozen commitments excluded automatically.

---

## Query Functions

- `getGarmentPercent(definitionId)` — sum of all netChange entries
- `getCommitmentWeeklyDeltas(definitionId, limit?)` — weekly delta summaries for display
- `getLiveWeekDelta(definitionId)` — running sum since Monday
- `getLiveWeekRatio()` — current week's partial ratio for progression projection

---

## Rules

- Subscribes to window domain events — never to `SchedulerTickEvent` directly
- The scheduler tick triggers `CommitmentSchedulerService` which publishes window events which trigger this service — three clean layers
- Publishes events after every significant state change
- All sub-calculations are pure functions
- `processWindowClose` is idempotent

---

## Dependencies

- `EventBus` — subscribes to window events, publishes performance events
- `ActivityService` — reads closed instance result
- `CommitmentService` — reads definition data
- `PerformanceRepository` — appends entries, reads entry history