**File Name**: service_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains the streak state for each commitment. Tracks consecutive kept and missed windows using a signed integer. Evaluates daily and specific-day commitments on window close, and weekly commitments at week end. Publishes `StreakChangedEvent` when a meaningful threshold is crossed.

`StreakRecord` does not implement `Achievable` — a streak is continuous state, not a discrete achievement moment. Milestone moments are detected by `MilestoneService`, which subscribes to `StreakChangedEvent`.

---

## Design Decisions

**Why Streak is its own feature.** Streak data is consumed by multiple features above it for different purposes. Keeping Streak standalone means each consumer subscribes to `StreakChangedEvent` or calls `getStreakRecord()` independently — no consumer depends on another.

**Why a signed integer.** A single signed integer carries both directions. Positive = consecutive kept windows, negative = consecutive missed windows, zero = neutral. Eliminates two separate counters and makes transition rules trivially simple.

**Why zero is a neutral state.** A positive streak that breaks goes to zero before going negative on the next miss. One missed window after a long kept streak is not immediately a bad streak — the pattern of missing is what matters.

**Why publish only above `minStreakDays` threshold.** Single-window changes are noise. Below the threshold the record updates silently — no event, no downstream reaction.

**Why weekly commitments are evaluated differently from daily ones.**

Consider a commitment to walk 3 times a week. The system generates one instance per day, each carrying 1/7 of the weekly target. If the user walks on Monday, Wednesday, and Friday — they meet their weekly target. But if the Tuesday instance closes with no log, that is not a failure — the user still has three more days to complete their 3-walk goal.

If streak were evaluated on every daily instance close, the Tuesday miss would decrement the streak, even though the week is not over and the user may still succeed. The daily instance close is not the right unit of accountability for a weekly commitment.

The correct unit is the week. At week end, look at the overall weekly score — did the user achieve their weekly target? That single question determines whether the streak increments or decrements. The instance carries `recurrence` in its snapshot, so `StreakService` knows at window close whether to act immediately or wait for the week boundary.

---

## AppConfig Constants

```
minStreakDays: 3
  Minimum abs(currentStreak) before StreakChangedEvent is published.
```

---

## Events Subscribed

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

First, check recurrence from the snapshot:

```
if snapshot.recurrence == Weekly: return   // weekly evaluated at week end, not here
if snapshot.commitmentState == frozen: return   // frozen windows are neutral
```

For Daily and SpecificWeekDays:

```
record = _getOrCreateRecord(event.definitionId)
record = _updateStreak(record, PerformanceService.isWindowSuccess(snapshot.livePerformance))
_saveAndPublish(record, snapshot.name, snapshot.commitmentState)
```

The `recurrence` and `commitmentState` are read directly from `event.snapshot` — no additional service calls needed. The instance carries everything required.

### `TemporalHelperService.onWeekEnded` → `_onWeekEnded(event)`

Evaluates streak for all Weekly commitments at week end.

```
instances = CommitmentIdentityService.getInstancesForWeek(event.weekStart)
weeklyInstances = instances.where(i => i.recurrence == Weekly)

for each unique definitionId in weeklyInstances:
  score = PerformanceService.getCommitmentWeekScore(definitionId, event.weekStart)
  record = _getOrCreateRecord(definitionId)
  record = _updateStreak(record, PerformanceService.isWindowSuccess(score))
  name = weeklyInstances.firstWhere(i => i.definitionId == definitionId).name
  _saveAndPublish(record, name, CommitmentState.active)
```

Weekly commitments that were frozen during the week are excluded — their instances carry `commitmentState == frozen`.

### `CommitmentService.onInstancePermanentlyDeleted` → `_onDeleted(event)`

Deletes the `StreakRecord` for this `definitionId`.

---

## Internal Functions

### `_updateStreak(record, isSuccess)` → StreakRecord

Pure function — no side effects.

```
if isSuccess:
  currentStreak = currentStreak >= 0 ? currentStreak + 1 : 0
else:
  currentStreak = currentStreak <= 0 ? currentStreak - 1 : 0

if currentStreak > bestStreak:
  bestStreak = currentStreak

return updated record
```

### `_saveAndPublish(record, commitmentName, commitmentState)`

Saves the record. Checks and updates the global best. Publishes events if thresholds are met.

```
StreakRepository.saveRecord(record)

// update global best if this is a new all-time high
if record.currentStreak > 0:
  globalBest = StreakRepository.getGlobalBest()
  if globalBest == null or record.bestStreak > globalBest.streakDays:
    StreakRepository.saveGlobalBest(GlobalBestStreakRecord(
      definitionId: record.definitionId,
      commitmentName: commitmentName,
      streakDays: record.bestStreak,
      achievedAt: now
    ))

// publish streak change if above threshold
if abs(record.currentStreak) >= AppConfig.minStreakDays:
  publish StreakChangedEvent(
    definitionId: record.definitionId,
    currentStreak: record.currentStreak,
    bestStreak: record.bestStreak
  )
```

### `_getOrCreateRecord(definitionId)` → StreakRecord

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreak: 0`.

---

## Events Published

```
StreakChangedEvent
  definitionId: String
  currentStreak: int     // signed — positive good, negative bad
  bestStreak: int
```

Published when `abs(currentStreak) >= AppConfig.minStreakDays`.

---

## Read Functions

### `getStreakRecord(definitionId)` → StreakRecord?

One-time read. Called by features above Streak that need current streak data for a specific commitment.

### `watchStreakRecord(definitionId)` → Stream<StreakRecord?>

Live stream. Used by the commitment detail screen for live streak display.

### `getBestStreakOverall()` → GlobalBestStreakRecord?

Returns the stored `GlobalBestStreakRecord`. Single-document read — no collection scan. Used by the Your Record screen.

---

## Rules

- `StreakRecord` does not implement `Achievable` — milestone detection belongs in `MilestoneService`
- `_updateStreak()` is a pure function — no side effects
- Weekly commitments skip `_onWindowClosed` — evaluated at week end only
- Frozen windows are neutral — streak record left unchanged
- `StreakChangedEvent` published only when `abs(currentStreak) >= AppConfig.minStreakDays`
- `bestStreak` tracks positive peaks only — never decreased by negative streaks
- `commitmentName` read from instance snapshot — no definition lookup needed
- `StreakService` never names its callers — it exposes its interface and stops there

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `TemporalHelperService` — subscribes to `onWeekEnded`
- `CommitmentIdentityService.getInstancesForWeek()` — fetches weekly instances for week-end evaluation
- `PerformanceService.isWindowSuccess()` — window classification
- `PerformanceService.getCommitmentWeekScore()` — weekly score for Weekly recurrence commitments
- `StreakRepository` — reads and writes `StreakRecord` and `GlobalBestStreakRecord`
- `AppConfig.minStreakDays`