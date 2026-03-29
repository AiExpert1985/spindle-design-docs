**File Name**: service_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** owns the complete lifecycle of garment profiles and day records. Initializes garments when commitments are created, updates them on performance changes, detects garment achievement moments, and exposes read functions to the presentation layer. The single writer of `GarmentProfile` and `GarmentDayRecord`.

---

## Abstractions

```
GarmentTypeResolver (abstract)
  → RuleBasedGarmentTypeResolver      // deterministic rules based on commitment shape
  → AIGarmentTypeResolver             // AI-assisted, Pro/Premium only (later)

ThreadColorResolver (abstract)
  → RandomVividColorResolver          // curated random palette
  → UserChosenColorResolver           // user picks colors (later)

GarmentDeltaCalculator (abstract)
  → DefaultGarmentDeltaCalculator     // contribution proportional to performance

GarmentRenderer (abstract)
  → SimpleThreadRenderer              // flat thread fill, visible strands
```

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onInstanceCreated(event)`

On a brand new commitment (no existing `GarmentProfile`):

1. `GarmentTypeResolver.resolve(definition)` → `garmentType`
2. `ThreadColorResolver.resolve(definitionId)` → `threadColors`
3. Create `GarmentProfile` — Do and Avoid both: `completionPercent: 0.0`

On every instance creation (new and recreated):

4. Snapshot current `AcceleratorService.getMultiplier(definitionId)` → `accelerationValue`
5. Create `GarmentDayRecord(definitionId, date: windowStart.date, accelerationValue, delta: 0.0)`

Idempotent — profile creation exits silently if profile already exists. Day record creation exits silently if a record already exists for this date.

### `PerformanceUpdatedEvent` → `_onPerformanceUpdated(event)`

```
1. Read GarmentDayRecord for (definitionId, windowStart.date)
2. if record == null: exit  // no record means no active instance — frozen, completed, or deleted
3. oldDelta = record.delta
4. performanceValue = _getPerformance(event.definitionId, event.livePerformance, record.accelerationValue)
5. newDelta = GarmentDeltaCalculator.calculate(performanceValue)
6. record.delta = newDelta
7. Save updated day record
8. profile.completionPercent = max(0.0, profile.completionPercent - oldDelta + newDelta)
9. Save updated profile
10. Publish GarmentUpdatedEvent
11. _detectAchievements(profile, profile.completionPercent + oldDelta - newDelta)
```

### `InstancePermanentlyDeletedEvent` → `_onCommitmentDeleted(event)`

Deletes `GarmentProfile` and all `GarmentDayRecord` records for this `definitionId`.

---

## Internal Functions

### `_getPerformance(definitionId, livePerformance, accelerationValue)` → double

```
if AppConfig.garmentUsesAccelerator == false:
  return livePerformance
return accelerationValue * livePerformance
```

### `_detectAchievements(profile, previousPercent)`

Checks all garment achievement thresholds. Called after every garment update with the previous `completionPercent`. Fires only on genuine upward threshold crossings.

```
// Each band crossing fires exactly once — on the upward transition only
thresholds = [100.0, 200.0, 300.0, 400.0]
subtypes   = [garmentCompleted, garmentFortified, garmentGolded, garmentDiamond]

for each (threshold, subtype) in zip(thresholds, subtypes):
  if previousPercent < threshold and profile.completionPercent >= threshold:
    _addAchievement(subtype, profile)
```

Adding a new phase means adding one threshold and one subtype — nothing else changes.

### `_addAchievement(subtype, profile)`

```
AchievementService.addAchievement(AchievementRecord(
  type: AchievementType.garment,
  subtype: subtype,
  sourceId: profile.id,
  definitionId: profile.definitionId,
  createdAt: now,
  updatedAt: now,
))
```

---

## Events Published

```
GarmentUpdatedEvent
  definitionId: String
  completionPercent: double
```

Published after every garment update. Consumed by the presentation layer for live display.

---

## Read Functions

### `watchGarmentProfile(definitionId)` → Stream<GarmentProfile?>

---

## Rules

- The only writer of `GarmentProfile` and `GarmentDayRecord`
- Reads definition once at garment creation — for `GarmentTypeResolver` input only
- All resolver/calculator abstractions are injected — never instantiated inside the service
- `accelerationValue` used for delta calculation always comes from the day record snapshot — never from the live accelerator
- `completionPercent` has no upper bound — lower bound is 0.0 only
- `_detectAchievements` fires only on genuine upward threshold crossings — idempotent by design
- `GarmentRenderer` used only by the garment display component — never called here

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstanceCreatedEvent`, `InstancePermanentlyDeletedEvent`
- `PerformanceService` — subscribes to `PerformanceUpdatedEvent`
- `AchievementService` — calls `addAchievement()`
- `AcceleratorService` — calls `getMultiplier()` at instance creation to snapshot acceleration
- `GarmentRepository` — reads and writes `GarmentProfile` and `GarmentDayRecord`
- `CommitmentService.getDefinition()` — reads definition once at garment creation
- `GarmentTypeResolver`, `ThreadColorResolver`, `GarmentDeltaCalculator` — injected