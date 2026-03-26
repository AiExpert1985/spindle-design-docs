**File Name**: service_accelerator **Feature**: Garment **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** maintains the acceleration multiplier for each commitment's garment. Reads streak data from `StreakService` — an independent feature below Garment in the dependency chain — when a window closes and adjusts the multiplier based on the current streak. Exposes one function that `GarmentService` calls to get the current multiplier. Internal to the Garment feature.

---

## Design Decisions

**Why AcceleratorService calls StreakService directly rather than AchievementService.** `StreakService` is an independent feature below Garment — a valid downward call. `AchievementService` sits at the same level as Garment — a same-level dependency, which is a violation. Garment reads streak data from Streak directly via `AcceleratorService`.

**Why the accelerator lives in Garment, not Streak.** The accelerator is a Garment concern — it interprets streak data to modify garment growth speed. Streak defines and measures streaks. What those numbers mean for the garment is Garment's decision.

**Why the adjustment formula is deferred.** The structure is ready: one multiplier per commitment, `_adjustMultiplier()` is the single function that changes it. The formula inside is the only thing unspecified — deliberately left open until real user data is available.

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

Called by `GarmentService._getPerformance()` on window close.

```
if AppConfig.garmentUsesAccelerator == false:
  return 1.0

streakRecord = StreakService.getStreakRecord(definitionId)
if streakRecord == null:
  return 1.0

record = _getOrCreateRecord(definitionId)
record = _adjustMultiplier(record, streakRecord.currentStreak)
saveRecord(record)

return record.multiplierValue
```

---

## Pure Functions

### `_adjustMultiplier(record, currentStreak)` → AcceleratorRecord

```
// Formula deferred — to be designed from real usage data.
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
- Calls `StreakService` directly — valid downward call since Streak sits below Garment
- Never calls `AchievementService` — same-level dependency violation
- `getMultiplier()` returns 1.0 when `garmentUsesAccelerator` is false
- `_adjustMultiplier()` is a pure function — no side effects

---

## Dependencies

- `StreakService.getStreakRecord()` — current streak per commitment
- `AcceleratorRepository` — reads and writes `AcceleratorRecord`
- `AppConfig` — `acceleratorFloor`, `acceleratorCeiling`, `garmentUsesAccelerator`
- EventBus — subscribes to `InstancePermanentlyDeletedEvent`

---

## Later Improvements

**Adjustment formula.** Design after real user data is available. The `_adjustMultiplier()` function is the single place to implement it.

**Asymmetric recovery rate.** Make the accelerator recover faster than it declines. Requires separate `acceleratorUpStep` and `acceleratorDownStep` constants in `AppConfig`.