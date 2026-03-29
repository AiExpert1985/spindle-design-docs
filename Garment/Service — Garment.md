**File Name**: service_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** owns the complete lifecycle of garment profiles and day records. Initializes garments when commitments are created, updates them on performance changes, detects garment achievement moments, and exposes read functions to the presentation layer. The single writer of `GarmentProfile`, `GarmentDayRecord`, and `CommitmentWeeklyProgress`.

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
3. Create `GarmentProfile` — Do: `completionPercent: 0.0`, Avoid: `completionPercent: 100.0`

On every instance creation (new and recreated):

4. Snapshot current `AcceleratorService.getMultiplier(definitionId)` → `accelerationValue`
5. Create `GarmentDayRecord(definitionId, date: windowStart.date, accelerationValue, delta: 0.0)`

Idempotent — profile creation exits silently if profile already exists. Day record creation exits silently if a record already exists for this date.

### `PerformanceUpdatedEvent` → `_onPerformanceUpdated(event)`

```
1. Read GarmentDayRecord for (definitionId, windowStart.date)
2. if record == null: exit  // safety — should not occur
3. if snapshot.commitmentState == frozen: exit  // safety net
4. oldDelta = record.delta
5. performanceValue = _getPerformance(event.definitionId, event.livePerformance, record.accelerationValue)
6. newDelta = GarmentDeltaCalculator.calculate(performanceValue, commitmentType)
7. record.delta = newDelta
8. Save updated day record
9. profile.completionPercent = clamp(profile.completionPercent - oldDelta + newDelta, 0.0, 100.0)
10. Save updated profile
11. Update live CommitmentWeeklyProgress record
12. Publish GarmentUpdatedEvent
13. _detectAchievements(profile, profile.completionPercent + oldDelta - newDelta)
```

### `InstancePermanentlyDeletedEvent` → `_onCommitmentDeleted(event)`

Deletes `GarmentProfile`, all `GarmentDayRecord` records, and all `CommitmentWeeklyProgress` records for this `definitionId`.

### `WeekEndedEvent` → `_onWeekEnded(event)`

Seals the live `CommitmentWeeklyProgress` record — sets `isCurrentWeek: false`. Creates new live record for the coming week.

---

## Internal Functions

### `_getPerformance(definitionId, livePerformance, accelerationValue)` → double

```
if AppConfig.garmentUsesAccelerator == false:
  return livePerformance
return accelerationValue * livePerformance
```

Uses the snapshotted `accelerationValue` from the day record — not the live accelerator value.

### `_detectAchievements(profile, previousPercent)`

Checks all garment achievement thresholds. Called after every garment update with the previous `completionPercent` before this update. Fires only on genuine threshold crossings.

```
isComplete = profile.completionPercent >= 100.0 (Do) or <= 0.0 (Avoid)
wasComplete = previousPercent >= 100.0 (Do) or <= 0.0 (Avoid)

if isComplete and not wasComplete:
  _addAchievement(AchievementSubtype.garmentCompleted, profile)
```

Adding a new achievement threshold means one new condition check here — nothing else changes.

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
  weeklyDelta: double
```

Published after every garment update. Consumed by the presentation layer for live display.

---

## Read Functions

### `watchGarmentProfile(definitionId)` → Stream<GarmentProfile?>

Live stream for the commitment detail screen.

### `getWeeklyProgress(definitionId, limit?)` → List<CommitmentWeeklyProgress>

Current week first.

### `getLiveWeekDelta(definitionId)` → double

Running delta for the current week.

---

## Rules

- The only writer of `GarmentProfile`, `GarmentDayRecord`, and `CommitmentWeeklyProgress`
- Reads definition once at garment creation — for `GarmentTypeResolver` input only
- All resolver/calculator abstractions are injected — never instantiated inside the service
- `accelerationValue` used for delta calculation always comes from the day record snapshot — never from the live accelerator
- `_detectAchievements` fires only on genuine threshold crossings — idempotent by design
- `GarmentRenderer` used only by the garment display component — never called here

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstanceCreatedEvent`, `InstancePermanentlyDeletedEvent`
- `PerformanceService` — subscribes to `PerformanceUpdatedEvent`
- `TemporalHelperService` — subscribes to `WeekEndedEvent`
- `AchievementService` — calls `addAchievement()`
- `AcceleratorService` — calls `getMultiplier()` at instance creation to snapshot acceleration
- `GarmentRepository` — reads and writes all three models
- `CommitmentService.getDefinition()` — reads definition once at garment creation
- `GarmentTypeResolver`, `ThreadColorResolver`, `GarmentDeltaCalculator` — injected