**File Name**: service_performance **Feature**: Performance **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 30-Mar-2026

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

Core formula. Returns `livePerformance` as a percentage (0–100+). Never called by other features.

**Do commitments:**

```
livePerformance = (totalLogged / currentTarget) × 100
```

No upper cap. Logging above target records the real value — 150% means the user genuinely overperformed. This preserves real data and signals when a target may be too easy. A done/not-done commitment uses `currentTarget = 1` — logging 1 gives 100%.

**Avoid commitments:**

When `totalLogged <= currentTarget` (user stayed within limit): `livePerformance = 100`.

When `totalLogged > currentTarget` and `currentTarget > 0` (user exceeded limit): soft exponential penalty with configurable base.

```
excess = (totalLogged - currentTarget) / currentTarget
livePerformance = 100 × AppConfig.avoidPenaltyBase ^ excess
```

Every doubling of excess halves the score at the default base of 0.5. The base is tunable — raising it toward 1.0 softens the penalty without touching the formula or any callers.

When `currentTarget == 0` (total avoidance — any logged amount is a breach): `livePerformance = 0`. No partial credit.

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

### `getWindowProgress(definitionId) → WindowProgress?`

Returns a ready-to-display progress snapshot for the current occurrence window. Returns null if no pending instance exists (frozen, completed, or between windows).

```
WindowProgress
  logged: double          // raw amount logged so far in this window
  target: double          // the target for this window
  unit: String?           // display unit — null if none set
  periodLabel: String     // "Today" or "This week"
  commitmentType: CommitmentType
```

```
instance = CommitmentIdentityService.getCurrentInstance(definitionId)
if instance == null: return null

entries = ActivityService.getEntriesForPeriod(
  definitionId,
  instance.regenerationWindow.windowStart,
  instance.regenerationWindow.windowEnd
)
logged = sum of entries.value

windowDays = instance.regenerationWindow.windowEnd
               .difference(instance.regenerationWindow.windowStart).inDays
periodLabel = windowDays == 1 ? "Today" : "This week"

return WindowProgress(
  logged: logged,
  target: instance.currentTarget.value,
  unit: instance.currentTarget.measureUnit,
  periodLabel: periodLabel,
  commitmentType: instance.commitmentType
)
```

The function is recurrence-agnostic — it never inspects `recurrence` type. The regeneration window boundaries already encode the correct accountability period for all recurrence types. The UI renders `"${periodLabel}: ${logged} of ${target}${unit ?? ''}"` with no further logic.

---

## Rules

- Two recalculation triggers only: `InstanceCreatedEvent` and `ActivityEvent`
- Never subscribes to `CommitmentEvent` or any event from a feature above Performance
- Recalculates only the affected instance per event — no bulk recalculation
- Instance status is never checked for recalculation — `livePerformance` updated regardless of pending or closed

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstanceCreatedEvent`; calls `updateLivePerformance()`, `getInstanceForCommitmentOnDate()`, `getInstances()`, `getCurrentInstance()`
- `ActivityService` — subscribes to `ActivityEvent`; calls `getTotalLoggedForCommitmentOnDate()`, `getEntriesForPeriod()`
- `AppConfig` — `successThreshold`, `avoidPenaltyBase`