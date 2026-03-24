**File Name**: performanceservice **Feature**: Performance **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** calculates and maintains `livePerformance` on commitment instances. Provides performance query functions to other features. No stored aggregates — all scores calculated on demand.

---

## Two Triggers

PerformanceService recalculates `livePerformance` in exactly two situations:

1. **`InstanceCreatedEvent`** — a new or recreated instance exists. Initialise `livePerformance` from any logs already recorded for that commitment on that date.
2. **`ActivityRecordedEvent`** — the user logged activity. Recalculate for the matching instance.

Everything that causes instance recreation — target changes, recurrence changes, state changes — is handled by CommitmentIdentityService, which publishes `InstanceCreatedEvent` after each recreation. PerformanceService never watches CommitmentEvent or definition changes directly.

---

## Writing livePerformance

After calculating, PerformanceService calls `CommitmentIdentityService.updateLivePerformance(instanceId, value)` directly. This is a valid downward call — Performance depends on Commitment, which is below it in the feature chain. See `feature_dependency_chain` for the full ordering.

`livePerformance` changes do not trigger `InstanceUpdatedEvent`. They are signalled separately via `PerformanceUpdatedEvent` so features that care about performance results subscribe to the right event without receiving noise from structural instance changes.

---

## Scoring Formulas

### Do Commitment

```
livePerformance = (totalLogged / target.value) × 100
```

No upper cap. Logging 150% records 150% — preserves real data and signals when a target is too easy. For day-level aggregates, individual contributions are capped at 100 before averaging.

A done/not-done commitment uses target = 1. Logging 1 = 100%.

### Avoid Commitment

Avoid commitments need a different approach from Do commitments. A Do commitment measures progress — more is better, the formula is linear. An Avoid commitment measures restraint — going over is failure, and going twice as far over should feel meaningfully worse than going slightly over.

**Rejected alternatives:**

`(target / logged) × 100` — the simple ratio. Smoking 5 against a limit of 1 gives 20%, which feels arbitrarily optimistic. The score compresses badly — a wide range of failures maps to a narrow band of low scores with no meaningful differentiation.

Linear penalty `max(0, 100 - excess × 100)` — floors at 0% the moment the user doubles their limit. Smoking 2 and smoking 20 both score 0%. Loses all information beyond the first doubling.

Pure exponential `(target / logged) ^ (logged / target)` — degrades too aggressively. 5× over gives 0.03%. Meaningful differentiation disappears above 2× excess.

**Chosen: soft exponential with configurable base.**

Continuous degradation with tunable steepness. Base 0.5 produces clean halving — every doubling of the excess halves the score. If the penalty proves too aggressive in real usage, raise the base toward 1.0 to soften it without touching the formula.

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

`livePerformance` stored as raw double — never rounded. Rounding is a display concern. The score drives visual representations (garment, arc, cup level) where the exact number is rarely shown directly.

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onInstanceCreated(event)`

New or recreated instance. Reads total logged for this commitment on this date. Calculates `livePerformance`. Calls `updateLivePerformance()`. Publishes `PerformanceUpdatedEvent`.

Fires for every instance generation — initial creation and every recreation after any definition change.

### `ActivityEvent` → `_onActivityEvent(event)`

A log was created, edited, or deleted. Finds the matching instance via `CommitmentIdentityService.getInstanceForCommitmentOnDate(definitionId, loggedAt)`. Reads total logged. Recalculates. Calls `updateLivePerformance()`. Publishes `PerformanceUpdatedEvent`.

The reaction is identical regardless of event type — PerformanceService always recalculates from the current total. It does not need to know whether a log was added, changed, or removed.

### `InstanceUpdatedEvent` → `_onInstanceUpdated(event)`

Checks `instance.status`. If `closed` — publishes `PerformanceUpdatedEvent(isClosed: true)` so downstream features react to the final result. No recalculation needed — `livePerformance` is already correct.

---

## Events Published

```
PerformanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  livePerformance: double
  isClosed: bool
```

Published after every `livePerformance` change and when a window closes. `isClosed: true` signals the final result.

Consumed by: Rewards (streak evaluation, `isClosed: true` only), Garment (every event), Encouragement (post-window feedback, `isClosed: true` only).

---

## Query Functions

Weekly scores are calculated on demand — no stored aggregate. Features needing weekly scores subscribe to `WeekEndedEvent` and call these functions themselves.

### `_calculate(totalLogged, currentTarget, commitmentType)` — private

Core formula. Never called by other features.

### `getPerformanceForPeriod(from, to, definitionId?)`

Universal reader. Fetches instances via `CommitmentIdentityService.getInstances(from, to, definitionId?)`. Returns average `livePerformance`. For day-level aggregates without `definitionId`, caps each instance at 100 before averaging.

### `getDayScore(date)`

`getPerformanceForPeriod(dayStart, dayEnd)` — performance calendar, day celebration, Your Record.

### `getCommitmentWeekScore(definitionId, weekStart)`

`getPerformanceForPeriod(weekStart, weekEnd, definitionId)` — commitment detail weekly view.

### `getOverallWeekScore(weekStart)`

`getPerformanceForPeriod(weekStart, weekEnd)` — weekly cup evaluation, progression gating, weekly summary.

---

## Rules

- Two recalculation triggers only: `InstanceCreatedEvent` and `ActivityRecordedEvent`
- Writes `livePerformance` via `CommitmentIdentityService.updateLivePerformance()` — a valid downward call
- Never subscribes to CommitmentEvent or any event from a feature above Performance
- Recalculates only the affected instance per event — no bulk recalculation
- No stored weekly aggregates — on demand only
- Instance status is never checked for recalculation — `livePerformance` updated regardless of pending or closed

---

## Dependencies

- EventBus — subscribes to InstanceCreatedEvent, InstanceUpdatedEvent, ActivityRecordedEvent; publishes PerformanceUpdatedEvent
- CommitmentIdentityService — updateLivePerformance(), getInstanceForCommitmentOnDate(), getInstances()
- ActivityService — getTotalLoggedForCommitmentOnDate()
- AppConfig — successThreshold (inactive in Phase 1)