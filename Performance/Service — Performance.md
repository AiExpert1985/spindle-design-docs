**File Name**: service_performance **Feature**: Performance **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** calculates and maintains `livePerformance` on commitment instances. Provides performance query functions to other features. No stored aggregates — all scores calculated on demand from instances and logs.

---

## Two Recalculation Triggers

`PerformanceService` recalculates `livePerformance` in exactly two situations:

1. **`InstanceCreatedEvent`** — a new or recreated instance exists. Reads any logs already recorded for that commitment on that date and calculates the initial `livePerformance`. This matters when an instance is recreated after a definition change — existing logs are not lost, and performance is immediately correct on recreation.
    
2. **`ActivityEvent`** — the user logged, edited, or deleted activity. Recalculates for the matching instance. The reaction is identical regardless of event type — always recalculate from the current total.
    

Everything that causes instance recreation — target changes, recurrence changes, state changes — is handled by `CommitmentIdentityService`, which publishes `InstanceCreatedEvent` after each recreation. `PerformanceService` never watches `CommitmentEvent` or definition changes directly.

---

## Writing livePerformance

After calculating, `PerformanceService` calls `CommitmentIdentityService.updateLivePerformance(instanceId, value)` directly. This is a valid downward call — Performance depends on Commitment, which is below it in the feature chain.

`livePerformance` changes do not trigger `InstanceUpdatedEvent`. They are signalled separately via `PerformanceUpdatedEvent` so features that care about performance results subscribe to the right event without receiving noise from structural instance changes.

---

## Scoring Formulas

### Do Commitment

```
livePerformance = (totalLogged / target.value) × 100
```

No upper cap — logging 150% records 150%. This preserves real data and signals when a target is too easy. Averages across periods are straight averages of raw `livePerformance` values.

A done/not-done commitment uses `target = 1`. Logging 1 = 100%.

### Avoid Commitment

Avoid commitments measure restraint — going over is failure, and going twice as far over should feel meaningfully worse than going slightly over.

**Rejected alternatives:**

`(target / logged) × 100` — the simple ratio. Smoking 5 against a limit of 1 gives 20%, which feels arbitrarily optimistic. The score compresses badly — a wide range of failures maps to a narrow band of low scores with no meaningful differentiation.

Linear penalty `max(0, 100 - excess × 100)` — floors at 0% the moment the user doubles their limit. Smoking 2 and smoking 20 both score 0%. Loses all information beyond the first doubling.

Pure exponential `(target / logged) ^ (logged / target)` — degrades too aggressively. 5× over gives 0.03%. Meaningful differentiation disappears above 2× excess.

**Chosen: soft exponential with configurable base.**

Continuous degradation with tunable steepness. Base 0.5 produces clean halving — every doubling of excess halves the score. If the penalty proves too aggressive in real usage, raise the base toward 1.0 to soften it without touching the formula.

```
target.value == 0 and totalLogged == 0  →  100%   (stayed clean)
target.value == 0 and totalLogged > 0   →  0%     (any slip is failure)

target.value > 0 and totalLogged <= target.value  →  100%
target.value > 0 and totalLogged > target.value   →
  excess = (totalLogged - target.value) / target.value
  score  = 100 × AppConfig.avoidPenaltyBase ^ excess
```

|target|logged|excess|score (base 0.5)|
|---|---|---|---|
|1|1|0|100% — on target|
|1|2|1|50% — doubled the limit|
|1|3|2|25% — tripled|
|1|4|3|12.5% — four times over|
|3|4|0.33|79.4% — slightly over|
|3|6|1|50% — doubled|
|3|9|2|25% — tripled|

`livePerformance` stored as raw double — never rounded. Rounding is a display concern.

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onInstanceCreated(event)`

New or recreated instance. Reads total logged for this commitment on this date via `ActivityService.getTotalLoggedForCommitmentOnDate()`. Calculates `livePerformance`. Calls `updateLivePerformance()`. Publishes `PerformanceUpdatedEvent`.

Fires for every instance generation — initial creation and every recreation after any definition change.

### `ActivityEvent` → `_onActivityEvent(event)`

A log was created, edited, or deleted. Finds the matching instance via `CommitmentIdentityService.getInstanceForCommitmentOnDate(definitionId, loggedAt)`. Reads total logged. Recalculates. Calls `updateLivePerformance()`. Publishes `PerformanceUpdatedEvent`.

The reaction is identical regardless of event type — `PerformanceService` always recalculates from the current total. It does not need to know whether a log was added, changed, or removed.

---

## Events Published

```
PerformanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  livePerformance: double
```

Published after every `livePerformance` change. Features that need to react to a closed window watch `InstanceUpdatedEvent` for `status: closed` directly — they do not need a separate closed signal from Performance.

---

## Query Functions

Weekly scores are calculated on demand — no stored aggregate. Features needing weekly scores subscribe to `WeekEndedEvent` and call these functions themselves.

### `_calculate(totalLogged, currentTarget, commitmentType)` — private

Core formula. Never called by other features.

### `getPerformanceForPeriod(from, to, definitionId?) → double`

Universal reader. Fetches instances via `CommitmentIdentityService.getInstances(from, to, definitionId?)`. Returns the straight average of `livePerformance` across all returned instances. No cap applied — raw values averaged as-is.

### `getDayScore(date) → double`

`getPerformanceForPeriod(dayStart, dayEnd)` — performance calendar, day celebration, Your Record.

### `getCommitmentWeekScore(definitionId, weekStart) → double`

`getPerformanceForPeriod(weekStart, weekEnd, definitionId)` — commitment detail weekly view.

### `getOverallWeekScore(weekStart) → double`

`getPerformanceForPeriod(weekStart, weekEnd)` — weekly cup evaluation, progression gating, weekly summary.

### `isWindowSuccess(livePerformance) → bool`

```
return livePerformance >= AppConfig.successThreshold
```

The single definition of what counts as a kept window. Any feature that needs to classify a window result as success or failure calls this function. Never reimplemented elsewhere. If `successThreshold` ever becomes per-commitment, only this function changes — all callers are unaffected.

---

## Rules

- Two recalculation triggers only: `InstanceCreatedEvent` and `ActivityEvent`
- Writes `livePerformance` via `CommitmentIdentityService.updateLivePerformance()` — a valid downward call
- Never subscribes to `CommitmentEvent` or any event from a feature above Performance
- Recalculates only the affected instance per event — no bulk recalculation
- No stored aggregates — all scores calculated on demand
- Instance status is never checked for recalculation — `livePerformance` updated regardless of pending or closed

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceCreatedEvent`
- `ActivityService` — subscribes to `ActivityEvent`; calls `getTotalLoggedForCommitmentOnDate()`
- `CommitmentIdentityService` — `updateLivePerformance()`, `getInstanceForCommitmentOnDate()`, `getInstances()`
- `AppConfig` — `successThreshold` (inactive in Phase 1), `avoidPenaltyBase`

---

## Later Improvements

**Performance caching.** As the app's history grows, `getPerformanceForPeriod` for a full month calendar fetches potentially hundreds of instances. A lightweight cache keyed by period and `definitionId` — invalidated on `PerformanceUpdatedEvent` — would eliminate redundant fetches without changing the public interface.