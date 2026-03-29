**File Name**: service_performance **Feature**: Performance **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** calculates and maintains `livePerformance` on commitment instances. Provides performance query functions to other features. No stored aggregates — all scores calculated on demand from instances and logs.

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onInstanceCreated(event)`

Reads total logged for this commitment on this date via `ActivityService.getTotalLoggedForCommitmentOnDate()`. Calculates `livePerformance`. Calls `updateLivePerformance()`. Publishes `PerformanceUpdatedEvent`.

### `ActivityEvent` → `_onActivityEvent(event)`

Finds the matching instance via `CommitmentIdentityService.getInstanceForCommitmentOnDate(definitionId, loggedAt)`. Reads total logged. Recalculates. Calls `updateLivePerformance()`. Publishes `PerformanceUpdatedEvent`.

---

## Events Published

```
PerformanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  livePerformance: double
```

Published after every `livePerformance` change.

---

## Query Functions

### `_calculate(totalLogged, currentTarget, commitmentType)` — private

Core formula. Never called by other features.

### `getPerformanceForPeriod(from, to, definitionId?) → double`

Fetches instances via `CommitmentIdentityService.getInstances(from, to, definitionId?)`. Returns the straight average of `livePerformance` across all returned instances. Returns 0.0 if no instances exist for the period. No cap applied — raw values averaged as-is.

### `getDayScore(date) → double`

`getPerformanceForPeriod(dayStart, dayEnd)`

### `getCommitmentWeekScore(definitionId, weekStart) → double`

`getPerformanceForPeriod(weekStart, weekEnd, definitionId)`

### `getOverallWeekScore(weekStart) → double`

`getPerformanceForPeriod(weekStart, weekEnd)`

### `isWindowSuccess(livePerformance) → bool`

```
return livePerformance >= AppConfig.successThreshold
```

If `successThreshold` ever becomes per-commitment, only this function changes — all callers are unaffected.

---

## Rules

- Two recalculation triggers only: `InstanceCreatedEvent` and `ActivityEvent`
- Never subscribes to `CommitmentEvent` or any event from a feature above Performance
- Recalculates only the affected instance per event — no bulk recalculation
- Instance status is never checked for recalculation — `livePerformance` updated regardless of pending or closed

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstanceCreatedEvent`; calls `updateLivePerformance()`, `getInstanceForCommitmentOnDate()`, `getInstances()`
- `ActivityService` — subscribes to `ActivityEvent`; calls `getTotalLoggedForCommitmentOnDate()`
- `AppConfig` — `successThreshold`, `avoidPenaltyBase`