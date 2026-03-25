**File Name**: service_accelerator **Feature**: Garment **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** maintains the acceleration multiplier for each commitment's garment. Reads streak data from `AchievementService` when a window closes and adjusts the multiplier based on the current streak. Exposes one function that `GarmentService` calls to get the current multiplier. Internal to the Garment feature.

---

## Design Decisions

**Why the accelerator reads from AchievementService instead of subscribing to StreakChangedEvent.** `StreakChangedEvent` is internal to the Achievements feature — not published externally. `AcceleratorService` reads streak state synchronously when `GarmentService` calls `_getPerformance()` on window close. This is the right trigger — the accelerator is needed exactly when the garment delta is calculated, and `PerformanceUpdatedEvent(isClosed: true)` is already the trigger for that. No separate subscription needed.

**Why the accelerator lives in Garment, not Achievements.** The accelerator is a Garment concern — it interprets streak data to modify garment growth speed. Streak defines and measures streaks. What those numbers mean for the garment is Garment's decision. This keeps Achievements as a pure recognition system with no knowledge of garment mechanics.

**Why the adjustment formula is deferred.** The structure is ready: `AcceleratorRecord` holds one multiplier, `_adjustMultiplier()` is the single function that changes it. The formula inside is the only thing unspecified — deliberately left open until real user data is available to tune it.

---

## AppConfig Constants

```
garmentUsesAccelerator: true
  Master switch. When false, getMultiplier() always returns 1.0.

acceleratorFloor: 0.75
  Minimum multiplier. Garment growth never slows below this.

acceleratorCeiling: 1.50
  Maximum multiplier. Asymmetric — momentum builds higher than it falls.
```

---

## Public Interface

### `getMultiplier(definitionId)` → double

Called by `GarmentService._getPerformance()` when a window closes.

```
if AppConfig.garmentUsesAccelerator == false:
  return 1.0

streakRecord = AchievementService.getStreakRecord(definitionId)
if streakRecord == null:
  return 1.0

record = _getOrCreateRecord(definitionId)
record = _adjustMultiplier(record, streakRecord.currentStreak)
saveRecord(record)

return record.multiplierValue
```

Reads the current streak, adjusts the multiplier, saves, and returns — all in one call. `GarmentService` receives the multiplier and applies it without knowing how it was derived.

---

## Pure Functions

### `_adjustMultiplier(record, currentStreak)` → AcceleratorRecord

Adjusts `multiplierValue` based on the streak direction and magnitude. Formula deferred.

```
// Formula to be designed based on real usage data.
// Inputs: record.multiplierValue, currentStreak (signed)
// Positive currentStreak → increase multiplier
// Negative currentStreak → decrease multiplier
// Result clamped to [acceleratorFloor, acceleratorCeiling]

newValue = _formula(record.multiplierValue, currentStreak)
record.multiplierValue = newValue.clamp(AppConfig.acceleratorFloor, AppConfig.acceleratorCeiling)
return record
```

### `_getOrCreateRecord(definitionId)` → AcceleratorRecord

Reads from repository. If null, creates with `multiplierValue: 1.0`.

---

## Events Subscribed

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Deletes the `AcceleratorRecord` for this `definitionId`.

---

## Rules

- Internal to Garment — no feature outside Garment calls this service
- `getMultiplier()` returns 1.0 when `garmentUsesAccelerator` is false
- `_adjustMultiplier()` is a pure function — no side effects
- `multiplierValue` always clamped to `[acceleratorFloor, acceleratorCeiling]`
- Adjustment formula is the only unspecified part — all surrounding structure is settled

---

## Dependencies

- `AchievementService.getStreakRecord()` — reads current streak on each window close
- `AcceleratorRepository` — reads and writes `AcceleratorRecord`
- `AppConfig` — `acceleratorFloor`, `acceleratorCeiling`, `garmentUsesAccelerator`
- EventBus — subscribes to `InstancePermanentlyDeletedEvent`

---

## Later Improvements

**Adjustment formula.** Design after real user data is available. Candidates: linear increment per streak unit, stepped at threshold values, exponential curve. The `_adjustMultiplier()` function is the single place to implement it.

**Asymmetric recovery rate.** Make the accelerator recover faster than it declines — reflecting that restarting a habit is easier than starting from scratch. Requires separate up/down step constants in `AppConfig`.