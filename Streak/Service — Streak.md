**File Name**: service_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains the streak state for each commitment using a signed integer. Tracks consecutive kept and missed windows. Publishes `StreakChangedEvent` when a meaningful threshold is crossed. Knows nothing about achievements, garments, or rewards — it only defines and measures streaks.

`StreakRecord` does not implement `Achievable` — a streak is continuous state, not a discrete achievement moment. Milestone moments are detected by `MilestoneService` inside the Achievements feature, which subscribes to `StreakChangedEvent`.

---

## Design Decisions

**Why a signed integer.** A single signed integer carries both directions. Positive = consecutive kept days, negative = consecutive missed days, zero = neutral. Eliminates two separate counters and makes transition rules trivially simple.

**Why zero is a neutral state.** A positive streak that breaks goes to zero before going negative on the next miss. One missed day after a long kept streak is not immediately a bad streak — the pattern of missing is what matters.

**Why publish only above `minStreakDays` threshold.** Single-day changes are noise. Below the threshold the record updates silently — no event, no downstream reaction.

**Why watch `InstanceUpdatedEvent` for closed windows, not `PerformanceUpdatedEvent`.** A streak update is triggered by a window closing — that is a lifecycle event on the instance, not a performance calculation event. `InstanceUpdatedEvent` with `status: closed` is the correct signal. The `livePerformance` value needed for streak classification is carried on the instance directly — no separate performance event needed.

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
record = _getOrCreateRecord(event.definitionId)
record = _updateStreak(record, event.snapshot.livePerformance)
saveRecord(record)

if abs(record.currentStreak) >= AppConfig.minStreakDays:
  publish StreakChangedEvent(record)

if record.currentStreak > 0 and isNewGlobalBest(record):
  publish GlobalBestStreakEvent(newBest, definitionId, commitmentName)
```

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Deletes the `StreakRecord` for this `definitionId`.

---

## Events Published

```
StreakChangedEvent
  definitionId: String
  currentStreak: int     // signed — positive good, negative bad
  bestStreak: int

GlobalBestStreakEvent
  newBest: int
  definitionId: String
  commitmentName: String
```

---

## Pure Functions

### `_updateStreak(record, livePerformance)` → StreakRecord

```
success = PerformanceService.isWindowSuccess(livePerformance)

if success:
  currentStreak = currentStreak >= 0 ? currentStreak + 1 : 0
else:
  currentStreak = currentStreak <= 0 ? currentStreak - 1 : 0

if currentStreak > bestStreak:
  bestStreak = currentStreak

return updated record
```

### `_getOrCreateRecord(definitionId)` → StreakRecord

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreak: 0`.

---

## Read Functions

### `getStreakRecord(definitionId)` → StreakRecord?

One-time read. Called by `AcceleratorService` (Garment) and `AchievementService`.

### `watchStreakRecord(definitionId)` → Stream<StreakRecord?>

Used by the commitment detail screen and `component_streak_ui`.

### `getBestStreakOverall()` → int

Returns the highest `bestStreak` across all records. Used by the Your Record screen.

---

## Rules

- `StreakRecord` does not implement `Achievable` — milestone moments are detected by `MilestoneService` in Achievements
- `_updateStreak()` is a pure function — no side effects
- Publishes only when `abs(currentStreak) >= AppConfig.minStreakDays`
- `bestStreak` tracks positive peaks only

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `StreakRepository` — reads and writes `StreakRecord`
- `PerformanceService.isWindowSuccess()` — window classification
- `AppConfig.minStreakDays`