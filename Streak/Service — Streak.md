**File Name**: service_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** maintains the streak state for each commitment. Evaluates daily and specific-day commitments on window close, and weekly commitments at week end. Detects per-commitment milestone achievements and the global best streak achievement directly — no separate milestone feature needed.

---

## AppConfig Constants

```
streakMilestones: every positive multiple of 3 (3, 6, 9, 12...)
  Handled by formula: currentStreak % 3 == 0 and currentStreak > 0
  No config list needed — formula handles any streak length automatically.
```

---

## Events Subscribed

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

```
if snapshot.recurrence == Weekly: return   // weekly evaluated at week end
if snapshot.commitmentState == frozen: return

record = _getOrCreateRecord(event.definitionId)
previousStreak = record.currentStreak
record = _updateStreak(record, PerformanceService.isWindowSuccess(snapshot.livePerformance))
StreakRepository.saveRecord(record)
_detectAchievements(record, previousStreak, snapshot.name)
```

### `WeekEndedEvent` → `_onWeekEnded(event)`

```
instances = CommitmentIdentityService.getInstancesForWeek(event.weekStart)
weeklyInstances = instances.where(i => i.recurrence == Weekly)

for each unique definitionId in weeklyInstances:
  score = PerformanceService.getCommitmentWeekScore(definitionId, event.weekStart)
  record = _getOrCreateRecord(definitionId)
  previousStreak = record.currentStreak
  record = _updateStreak(record, PerformanceService.isWindowSuccess(score))
  name = weeklyInstances.first(definitionId).name
  StreakRepository.saveRecord(record)
  _detectAchievements(record, previousStreak, name)
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

if currentStreak > bestStreak:
  bestStreak = currentStreak

return updated record
```

### `_detectAchievements(record, previousStreak, commitmentName)`

```
// per-commitment milestone — every positive multiple of 3
if record.currentStreak > 0 and record.currentStreak % 3 == 0
    and record.currentStreak != previousStreak:
  _addAchievement(AchievementSubtype.streakMilestone, record.definitionId, commitmentName)

// global best — only when strictly greater than all others
if record.bestStreak > previousBestStreak(record):
  globalBest = _findBestStreak(excluding: record.definitionId)
  if record.bestStreak > globalBest:
    _addAchievement(AchievementSubtype.globalBestStreak, record.definitionId, commitmentName)
```

### `_findBestStreak(excluding definitionId?)` → int

Fetches all `StreakRecord`s, optionally excluding one `definitionId`, and returns the highest `bestStreak` value. At most a few dozen records per user — not expensive.

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

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreak: 0`.

---

## Read Functions

### `getStreakRecord(definitionId)` → StreakRecord?

One-time read. Called by features above Streak that need current streak state.

### `watchStreakRecord(definitionId)` → Stream<StreakRecord?>

Live stream for UI display.

### `getBestStreakOverall()` → StreakRecord?

Returns the `StreakRecord` with the highest `bestStreak` across all commitments. Computed via `_findBestStreak()`.

---

## Rules

- No `StreakChangedEvent` — streak changes are internal
- Weekly commitments skip `_onWindowClosed` — evaluated at week end only
- Frozen windows are neutral — record left unchanged
- `bestStreak` tracks positive peaks only
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
- `AppConfig` — streak milestone formula constants