**File Name**: service_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains the streak state for each commitment. Tracks consecutive kept and missed windows using a signed integer. Evaluates daily and specific-day commitments on window close, and weekly commitments at week end. Publishes `StreakChangedEvent` when a meaningful threshold is crossed. Calls `AchievementService.addAchievement()` when a new global best crosses a milestone threshold.

`StreakRecord` is continuous state — not a discrete achievement moment. Milestone moments are detected by `MilestoneService`, which subscribes to `StreakChangedEvent`.

---

## Design Decisions

**Why Streak is its own feature.** Streak data is consumed by multiple features above it for different purposes. Keeping Streak standalone means each consumer subscribes to `StreakChangedEvent` or calls `getStreakRecord()` independently.

**Why a signed integer.** Positive = consecutive kept windows, negative = consecutive missed windows, zero = neutral. Eliminates two separate counters and makes transition rules trivially simple.

**Why zero is a neutral state.** A positive streak that breaks goes to zero before going negative. One missed window after a long kept streak is not immediately a bad streak — the pattern of missing is what matters.

**Why publish only above `minStreakDays` threshold.** Single-window changes are noise. Below the threshold the record updates silently.

**Why weekly commitments are evaluated differently from daily ones.** Consider "walk 3 times a week." The system generates one instance per day. If Tuesday closes with no log, that is not a failure — the user still has days left to complete their 3-walk goal. Evaluating the streak on every daily instance close would decrement it incorrectly. The week is the right unit. At week end, the overall weekly score determines success or failure for the whole week.

The instance carries `recurrence` in its snapshot — `StreakService` knows at window close whether to act immediately (Daily, SpecificWeekDays) or wait for week end (Weekly).

---

## AppConfig Constants

```
minStreakDays: 3
  Minimum abs(currentStreak) before StreakChangedEvent is published.
```

---

## Events Subscribed

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

```
if snapshot.recurrence == Weekly: return   // weekly evaluated at week end
if snapshot.commitmentState == frozen: return   // safety net

record = _getOrCreateRecord(event.definitionId)
record = _updateStreak(record, PerformanceService.isWindowSuccess(snapshot.livePerformance))
_saveAndPublish(record, snapshot.name)
```

### `TemporalHelperService.onWeekEnded` → `_onWeekEnded(event)`

```
instances = CommitmentIdentityService.getInstancesForWeek(event.weekStart)
weeklyInstances = instances.where(i => i.recurrence == Weekly)

for each unique definitionId in weeklyInstances:
  score = PerformanceService.getCommitmentWeekScore(definitionId, event.weekStart)
  record = _getOrCreateRecord(definitionId)
  record = _updateStreak(record, PerformanceService.isWindowSuccess(score))
  name = weeklyInstances.first(definitionId).name
  _saveAndPublish(record, name)
```

### `CommitmentService.onInstancePermanentlyDeleted` → `_onDeleted(event)`

Deletes the `StreakRecord` for this `definitionId`.

---

## Internal Functions

### `_updateStreak(record, isSuccess)` → StreakRecord

Pure function.

```
if isSuccess:
  currentStreak = currentStreak >= 0 ? currentStreak + 1 : 0
else:
  currentStreak = currentStreak <= 0 ? currentStreak - 1 : 0

if currentStreak > bestStreak:
  bestStreak = currentStreak

return updated record
```

### `_saveAndPublish(record, commitmentName)`

```
StreakRepository.saveRecord(record)

// update global best and call addAchievement if new best crosses a milestone
if record.currentStreak > 0:
  globalBest = StreakRepository.getGlobalBest()
  if globalBest == null or record.bestStreak > globalBest.streakDays:
    if AppConfig.streakMilestones.contains(record.bestStreak):
      newBest = GlobalBestStreakRecord(
        definitionId: record.definitionId,
        commitmentName: commitmentName,
        streakDays: record.bestStreak,
        achievedAt: now,
      )
      StreakRepository.saveGlobalBest(newBest)
      AchievementService.addAchievement(AchievementRecord(
        type: AchievementType.streak,
        subtype: AchievementSubtype.globalBestStreak,
        sourceId: record.definitionId,
        definitionId: record.definitionId,
        createdAt: now,
        updatedAt: now,
      ))

// publish streak change if above threshold
if abs(record.currentStreak) >= AppConfig.minStreakDays:
  publish StreakChangedEvent(
    definitionId: record.definitionId,
    currentStreak: record.currentStreak,
    bestStreak: record.bestStreak,
  )
```

**Why global best published only at milestone values:** publishing on every day of a long streak would flood the achievement history. At milestone values the achievement means something specific — the user has never kept any commitment this long before.

### `_getOrCreateRecord(definitionId)` → StreakRecord

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreak: 0`.

---

## Events Published

```
StreakChangedEvent
  definitionId: String
  currentStreak: int
  bestStreak: int
```

Published when `abs(currentStreak) >= AppConfig.minStreakDays`. Consumed by `MilestoneService`.

---

## Read Functions

### `getStreakRecord(definitionId)` → StreakRecord?

One-time read. Called by features above Streak that need current streak state.

### `watchStreakRecord(definitionId)` → Stream<StreakRecord?>

Live stream. Used by the commitment detail screen.

### `getBestStreakOverall()` → GlobalBestStreakRecord?

Returns the stored global best. Single-document read.

---

## Rules

- Weekly commitments skip `_onWindowClosed` — evaluated at week end only
- Frozen windows are neutral — streak record left unchanged
- `StreakChangedEvent` published only when `abs(currentStreak) >= AppConfig.minStreakDays`
- `bestStreak` tracks positive peaks only
- Global best achievement only at milestone values — prevents achievement dilution
- `commitmentName` read from instance snapshot — no definition lookup needed

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `TemporalHelperService` — subscribes to `onWeekEnded`
- `CommitmentIdentityService.getInstancesForWeek()` — weekly evaluation
- `PerformanceService.isWindowSuccess()` — window classification
- `PerformanceService.getCommitmentWeekScore()` — weekly score
- `AchievementService.addAchievement()` — records global best achievement
- `StreakRepository` — reads and writes `StreakRecord` and `GlobalBestStreakRecord`
- `AppConfig.minStreakDays`, `AppConfig.streakMilestones`