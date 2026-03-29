**File Name**: service_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** maintains the streak state for each commitment. Evaluates daily and specific-day commitments on window close, and weekly commitments at week end. Detects per-commitment milestone achievements and the global best streak achievement directly — no separate milestone feature needed.

---

## AppConfig Constants

```
streakStep: 3
  The interval between streak milestones.
  Milestone fires when currentStreak % streakStep == 0 and currentStreak > 0.
  Changing this value adjusts all milestone thresholds automatically.
```

---

## Events Subscribed

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

```
if snapshot.recurrence == Weekly: return   // weekly evaluated at week end
if snapshot.commitmentState == frozen: return

previousRecord = _getOrCreateRecord(event.definitionId)
updatedRecord = _updateStreak(previousRecord, PerformanceService.isWindowSuccess(snapshot.livePerformance))
StreakRepository.saveRecord(updatedRecord)
_detectAchievements(updatedRecord, previousRecord, snapshot.name)
```

### `WeekEndedEvent` → `_onWeekEnded(event)`

```
instances = CommitmentIdentityService.getInstancesForWeek(event.weekStart)
weeklyInstances = instances.where(i => i.recurrence == Weekly)

for each unique definitionId in weeklyInstances:
  score = PerformanceService.getCommitmentWeekScore(definitionId, event.weekStart)
  previousRecord = _getOrCreateRecord(definitionId)
  updatedRecord = _updateStreak(previousRecord, PerformanceService.isWindowSuccess(score))
  name = weeklyInstances.first(definitionId).name
  StreakRepository.saveRecord(updatedRecord)
  _detectAchievements(updatedRecord, previousRecord, name)
```

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

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

if currentStreak > bestStreakValue:
  bestStreakValue = currentStreak
  bestStreakDate = now

return updated record
```

### `_detectAchievements(record, previousRecord, commitmentName)`

```
// per-commitment milestone — every positive multiple of streakStep
if record.currentStreak > 0 and record.currentStreak % AppConfig.streakStep == 0
    and record.currentStreak != previousRecord.currentStreak:
  _addAchievement(AchievementSubtype.streakMilestone, record.definitionId, commitmentName)

// global best — only when bestStreakValue improved and is strictly greater than all others
if record.bestStreakValue > previousRecord.bestStreakValue:
  best = _findBestStreak(excluding: record.definitionId)
  if best == null or record.bestStreakValue > best.bestStreakValue:
    _addAchievement(AchievementSubtype.globalBestStreak, record.definitionId, commitmentName)
```

### `_findBestStreak(excluding definitionId?)` → StreakRecord?

Fetches all `StreakRecord`s, optionally excluding one `definitionId`, and returns the record with the highest `bestStreakValue`. Returns null if no records exist. At most a few dozen records per user.

### `_addAchievement(subtype, definitionId, commitmentName)`

```
AchievementService.addAchievement(AchievementRecord(
  type: AchievementType.streak,
  subtype: subtype,
  sourceId: definitionId,
  definitionId: definitionId,
  createdAt: now,
  updatedAt: now,
))
```

### `_getOrCreateRecord(definitionId)` → StreakRecord

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreakValue: 0`, `bestStreakDate: null`.

---

## Read Functions

### `getStreakRecord(definitionId)` → StreakRecord?

One-time read. Called by features above Streak that need current streak state.

### `watchStreakRecord(definitionId)` → Stream<StreakRecord?>

Live stream for UI display.

### `getBestStreakOverall()` → StreakRecord?

Returns the `StreakRecord` with the highest `bestStreakValue` across all commitments. Computed via `_findBestStreak()` with no exclusion. Returns null if no records exist yet.

---

## Rules

- No `StreakChangedEvent` — streak changes are internal
- Weekly commitments skip `_onWindowClosed` — evaluated at week end only
- Frozen windows are neutral — record left unchanged
- `bestStreakValue` and `bestStreakDate` updated together — only when a new positive best is reached
- Per-commitment milestone fires at every positive multiple of 3
- Global best fires only when strictly greater than all others — ties ignored
- `commitmentName` read from instance snapshot — no definition lookup needed

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `TemporalHelperService` — subscribes to `WeekEndedEvent`
- `CommitmentIdentityService.getInstancesForWeek()` — weekly evaluation
- `PerformanceService.isWindowSuccess()` — window classification
- `PerformanceService.getCommitmentWeekScore()` — weekly score
- `AchievementService.addAchievement()` — records achievements
- `StreakRepository` — reads and writes `StreakRecord`
- `AppConfig` — `streakStep`