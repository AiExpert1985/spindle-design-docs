**File Name**: service_accelerator **Feature**: Garment **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains the acceleration multiplier for each commitment's garment. Reads streak data from `StreakService` — an independent feature below Garment in the dependency chain — on each garment update and adjusts the multiplier based on the current streak. Exposes one function that `GarmentService` calls to get the current multiplier. Internal to the Garment feature.

---

## Design Decisions

**Why AcceleratorService calls StreakService directly rather than AchievementService.** `StreakService` is an independent feature below Garment — a valid downward call. `AchievementService` sits at the same level as Garment — a same-level dependency, which is a violation. Garment reads streak data from Streak directly via `AcceleratorService`.

**Why the accelerator lives in Garment, not Streak.** The accelerator is a Garment concern — it interprets streak data to modify garment growth speed. Streak defines and measures streaks. What those numbers mean for the garment is Garment's decision.

**Why the adjustment formula is deferred.** The structure is ready: one multiplier per commitment, `_adjustMultiplier()` is the single function that changes it. The formula inside is deliberately left open until real user data is available after launch.

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

Called by `GarmentService._getPerformance()` on each performance update.

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

The `AcceleratorRecord` is created lazily on first call for a given commitment — not on a subscription event. `_getOrCreateRecord` returns the existing record if one exists, or creates one with `multiplierValue: 1.0` if not.

---

## Pure Functions

### `_adjustMultiplier(record, currentStreak)` → AcceleratorRecord

Adjusts the multiplier based on the current streak value. Positive streak increases the multiplier toward `acceleratorCeiling`. Negative streak decreases it toward `acceleratorFloor`. Result clamped to `[acceleratorFloor, acceleratorCeiling]`.

The specific formula is deferred — it will be designed from real usage data after launch. `_adjustMultiplier()` is the single location where the formula lives.

### `_getOrCreateRecord(definitionId)` → AcceleratorRecord

Reads from repository. If null, creates with `multiplierValue: 1.0`.

---

## Events Subscribed

### `InstancePermanentlyDeletedEvent` (CommitmentIdentityService) → `_onDeleted(event)`

Deletes the `AcceleratorRecord` for this `definitionId`.

---

## Rules

- Internal to Garment — no feature outside Garment calls this service
- Calls `StreakService.getStreakRecord()` directly — valid downward call since Streak sits below Garment
- Never calls `AchievementService` — same-level dependency violation
- `getMultiplier()` returns 1.0 when `garmentUsesAccelerator` is false or no streak record exists
- `_adjustMultiplier()` is a pure function — no side effects
- Records created lazily on first `getMultiplier()` call — no separate initialization event

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstancePermanentlyDeletedEvent`
- `StreakService.getStreakRecord()` — current streak per commitment
- `AcceleratorRepository` — reads and writes `AcceleratorRecord`
- `AppConfig` — `acceleratorFloor`, `acceleratorCeiling`, `garmentUsesAccelerator`