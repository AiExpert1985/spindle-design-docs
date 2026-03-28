**File Name**: service_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains the streak state for each commitment. Tracks consecutive kept and missed windows using a signed integer. Evaluates daily and specific-day commitments on window close, and weekly commitments at week end. Records streak achievements directly via `AchievementService.addAchievement()` — no separate milestone feature needed.

---

## Design Decisions

**Why Streak is its own feature.** Streak data is consumed by multiple features above it for different purposes. Keeping Streak standalone means each consumer calls `getStreakRecord()` independently without depending on each other.

**Why a signed integer.** Positive = consecutive kept windows, negative = consecutive missed windows, zero = neutral. Eliminates two separate counters and makes transition rules trivially simple.

**Why zero is a neutral state.** A positive streak that breaks goes to zero before going negative on the next miss. One missed window after a long kept streak is not immediately a bad streak — the pattern of missing is what matters.

**Why streak milestone detection lives here, not in a separate Milestones feature.** The original Milestones feature existed only to watch `StreakChangedEvent` and translate threshold crossings into achievements. That was a legacy design. Now `StreakService` detects thresholds directly where it already has all the data — `currentStreak`, `definitionId`, `commitmentName`. Removing the middleman makes the flow explicit and eliminates an entire feature with no loss of functionality.

**Why there is no `StreakChangedEvent`.** `StreakChangedEvent` existed solely so `MilestoneService` could subscribe. With Milestones gone and achievement detection internal to this service, no external feature needs to subscribe to streak changes. The event is removed. Any feature that needs streak data calls `getStreakRecord()` directly.

**Why weekly commitments are evaluated at week end, not on daily window close.** Consider "walk 3 times a week." If Tuesday closes with no log, that is not a failure — the user still has days left. Evaluating on every daily close would incorrectly decrement the streak. The week is the right unit. The instance carries `recurrence` in its snapshot — `StreakService` checks it at window close and skips weekly commitments there.

---

## AppConfig Constants

```
minStreakThreshold: 3
  Minimum abs(currentStreak) before achievement detection runs.
  Below this, changes are too small to be meaningful.

streakMilestones: [3, 5, 7, 10, 14]
  The streak day counts that trigger a milestone achievement.
  Matches the badge mapping in component_streak_ui.
  Adjustable after launch — badge mapping must be updated to match.
```

---

## Events Subscribed

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

```
if snapshot.recurrence == Weekly: return   // weekly evaluated at week end
if snapshot.commitmentState == frozen: return   // safety net

record = _getOrCreateRecord(event.definitionId)
previousRecord = record.copy()
record = _updateStreak(record, PerformanceService.isWindowSuccess(snapshot.livePerformance))
StreakRepository.saveRecord(record)
_detectAchievements(record, previousRecord, snapshot.name)
```

### `TemporalHelperService.onWeekEnded` → `_onWeekEnded(event)`

```
instances = CommitmentIdentityService.getInstancesForWeek(event.weekStart)
weeklyInstances = instances.where(i => i.recurrence == Weekly)

for each unique definitionId in weeklyInstances:
  score = PerformanceService.getCommitmentWeekScore(definitionId, event.weekStart)
  record = _getOrCreateRecord(definitionId)
  previousRecord = record.copy()
  record = _updateStreak(record, PerformanceService.isWindowSuccess(score))
  name = weeklyInstances.first(definitionId).name
  StreakRepository.saveRecord(record)
  _detectAchievements(record, previousRecord, name)
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

### `_detectAchievements(record, previousRecord, commitmentName)`

Central achievement detection. Called after every streak update. Checks all conditions and calls `_addAchievement()` for each that is met. Adding a new streak achievement condition means adding one check here.

```
// milestone detection — positive streak crossing a threshold for the first time
if record.currentStreak > 0
    and AppConfig.streakMilestones.contains(record.currentStreak)
    and record.currentStreak != previousRecord.currentStreak:
  _addAchievement(
    subtype: _milestoneSubtype(record.currentStreak),
    definitionId: record.definitionId,
    commitmentName: commitmentName,
  )

// global best detection — new all-time best at a milestone value
if record.bestStreak > previousRecord.bestStreak
    and AppConfig.streakMilestones.contains(record.bestStreak):
  globalBest = StreakRepository.getGlobalBest()
  if globalBest == null or record.bestStreak > globalBest.streakDays:
    StreakRepository.saveGlobalBest(GlobalBestStreakRecord(
      definitionId: record.definitionId,
      commitmentName: commitmentName,
      streakDays: record.bestStreak,
      achievedAt: now,
    ))
    _addAchievement(
      subtype: AchievementSubtype.globalBestStreak,
      definitionId: record.definitionId,
      commitmentName: commitmentName,
    )
```

**Why global best is published only at milestone values:** publishing on every new best would flood the achievement history — a user on a 30-day streak earns a new personal best every day from day 15 onward. At milestone values the achievement means something specific: the user has never sustained any commitment this long before.

### `_addAchievement(subtype, definitionId, commitmentName)`

Private. Constructs and submits an `AchievementRecord`.

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

### `_milestoneSubtype(streakCount)` → AchievementSubtype

```
3  → threeDay
5  → fiveDay
7  → sevenDay
10 → tenDay
14 → fourteenDay
```

### `_getOrCreateRecord(definitionId)` → StreakRecord

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreak: 0`.

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

- No `StreakChangedEvent` — streak changes are internal. External features call `getStreakRecord()` for streak data.
- Weekly commitments skip `_onWindowClosed` — evaluated at week end only
- Frozen windows are neutral — record left unchanged
- `bestStreak` tracks positive peaks only
- Milestone achievements only at values in `AppConfig.streakMilestones`
- Global best achievement only when new best crosses a milestone value
- `commitmentName` read from instance snapshot — no definition lookup needed

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `TemporalHelperService` — subscribes to `onWeekEnded`
- `CommitmentIdentityService.getInstancesForWeek()` — weekly evaluation
- `PerformanceService.isWindowSuccess()` — window classification
- `PerformanceService.getCommitmentWeekScore()` — weekly score
- `AchievementService.addAchievement()` — records achievements
- `StreakRepository` — reads and writes `StreakRecord` and `GlobalBestStreakRecord`
- `AppConfig` — `minStreakThreshold`, `streakMilestones`