**File Name**: service_streak **Feature**: Achievements (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** maintains the streak state for each commitment. Internal to Achievements — never called directly by features outside. Publishes `StreakChangedEvent` as an internal event consumed by `MilestoneService` and `AcceleratorService` (Garment-internal) within their respective features. External features read streak data only through `AchievementService.getStreakRecord()`.

---

## Design Decisions

**Why streak is its own feature.** Streak data is consumed by multiple features for different purposes — Garment uses it for acceleration, Rewards uses it for milestone celebrations, Encouragement uses it for motivational moments, Analytics uses it for pattern analysis. Keeping streak as a standalone feature means each consumer subscribes to the same clean events without any of them needing to know about the others.

**Why a signed integer instead of two separate counters.** A single signed integer carries both directions with one field. Positive = good streak, negative = bad streak, zero = neutral. This eliminates `keptStreak` and `missedStreak` as separate fields and makes the transition rules trivially simple. The sign is the signal — subscribers read it and react accordingly.

**Why zero is a distinct neutral state.** A positive streak that breaks goes to zero before going negative on the next miss. One missed day after a long kept streak is not immediately a bad streak — it is a neutral moment. The bad streak signal only fires after the user has missed for `AppConfig.minStreakDays` consecutive days. This prevents false punishment for a single slip.

**Why publish only above `minStreakDays` threshold.** Single-day changes are noise. A streak only becomes meaningful — and worth signalling to other features — when it has persisted for a minimum number of consecutive days. Below that threshold, the service updates the record silently with no event. This keeps the event bus quiet during normal day-to-day fluctuation.

**Why one event instead of separate kept/broken/best events.** `StreakChangedEvent` carries the full current state — `currentStreak` (signed) and `bestStreak`. Subscribers read what they need. The sign tells them direction. The value tells them magnitude. A separate `StreakBestEvent` would be redundant — the subscriber can detect a new best by comparing `currentStreak` to `bestStreak` in the payload. One event, full information.

---

## AppConfig Constants

```
minStreakDays: 3
  Minimum absolute streak value before StreakChangedEvent is published.
  Below this threshold, the record updates silently.
  Prevents noise from single-day fluctuations.
```

---

## Events Subscribed

### `PerformanceUpdatedEvent` where `isClosed: true` → `_onWindowClosed(event)`

Fires when a commitment window closes with its final result.

```
record = _getOrCreateRecord(event.definitionId)
record = _updateStreak(record, event.livePerformance)
saveRecord(record)

if abs(record.currentStreak) >= AppConfig.minStreakDays:
  publish StreakChangedEvent(record)

if record.currentStreak > 0:
  globalBest = _checkGlobalBest(record)
  if globalBest changed:
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
  bestStreak: int        // all-time positive peak for this commitment
```

Published when `abs(currentStreak) >= AppConfig.minStreakDays`. The sign tells subscribers the direction. The value tells them the magnitude. Subscribers never need a follow-up call — the event carries the full current state.

```
GlobalBestStreakEvent
  newBest: int           // the new global best streak value
  definitionId: String   // which commitment set the new global best
  commitmentName: String // snapshotted at publish time — no follow-up lookup needed
```

Published when any commitment sets a new personal best that also exceeds the current global best across all commitments. Used by Rewards for a future global milestone reward. `commitmentName` is snapshotted so subscribers never need to call back for display data.

---

## Pure Functions

### `_updateStreak(record, livePerformance)` → StreakRecord

Applies the signed transition rules. Never saves.

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

The transition through zero on direction change is explicit — a positive streak breaking goes to 0, a negative streak recovering goes to 0. The next window result then takes it to +1 or -1.

### `_getOrCreateRecord(definitionId)` → StreakRecord

Reads from repository. If null, creates with `currentStreak: 0`, `bestStreak: 0`.

### `_checkGlobalBest(record)` → bool

Reads all records via `StreakRepository.getAllRecords()`. Returns true if `record.currentStreak` exceeds the highest `bestStreak` across all records.

---

## Read Functions

### `getStreakRecord(definitionId)` → StreakRecord?

One-time read. Used by AcceleratorService and the commitment detail screen.

### `watchStreakRecord(definitionId)` → Stream<StreakRecord?>

Stream. Used by the commitment detail screen for live streak display.

### `getBestStreakOverall()` → int

Returns the highest `bestStreak` value across all records. Used by the Your Record screen.

---

## Rules

- The only writer of `StreakRecord`
- `CommitmentDefinition` has no streak fields
- Frozen windows produce no `PerformanceUpdatedEvent` — streak is preserved through freeze automatically
- `_updateStreak()` is a pure function — never saves, never calls any service
- Publishes only when `abs(currentStreak) >= AppConfig.minStreakDays`
- `bestStreak` tracks positive peaks only — negative streaks never affect it
- `GlobalBestStreakEvent` published only when a new commitment-level best also exceeds the global best

---

## Dependencies

- EventBus — subscribes to `PerformanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`; publishes `StreakChangedEvent`, `GlobalBestStreakEvent`
- `StreakRepository` — reads and writes `StreakRecord`
- `PerformanceService.isWindowSuccess()` — single source of truth for window success classification
- `AppConfig` — `minStreakDays`