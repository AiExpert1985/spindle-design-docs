**File Name**: service_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** owns the complete lifecycle of garment profiles. Initializes garments when commitments are created, updates them live as performance changes, detects garment achievement moments, and exposes read functions to the presentation layer. The single writer of `GarmentProfile` and `CommitmentWeeklyProgress`.

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
  → DefaultGarmentDeltaCalculator     // contribution + decay + streak bonus

GarmentRenderer (abstract)
  → SimpleThreadRenderer              // flat thread fill, visible strands
```

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onCommitmentCreated(event)`

Creates a `GarmentProfile` for this commitment:

1. `GarmentTypeResolver.resolve(definition)` → `garmentType`
2. `ThreadColorResolver.resolve(definitionId)` → `threadColors`
3. Creates profile — Do: `completionPercent: 0.0`, Avoid: `completionPercent: 100.0`

Idempotent — exits silently if profile already exists.

### `PerformanceUpdatedEvent` → `_onPerformanceUpdated(event)`

```
1. Read current GarmentProfile
2. Check lastUpdatedDate — if already updated today, exit (idempotent)
3. if snapshot.commitmentState == frozen: exit  // safety net
4. previousPercent = profile.completionPercent
5. performanceValue = _getPerformance(event.definitionId, event.livePerformance)
6. isSuccess = PerformanceService.isWindowSuccess(event.livePerformance)
7. delta = GarmentDeltaCalculator.calculate(profile, performanceValue, isSuccess)
8. Apply delta — clamp completionPercent to 0.0–100.0
9. Save updated profile
10. Update live CommitmentWeeklyProgress record
11. Publish GarmentUpdatedEvent
12. _detectAchievements(profile, previousPercent)
```

**Why the frozen check is a safety net:** `PerformanceUpdatedEvent` only fires from `InstanceCreatedEvent` and `ActivityEvent` — neither fires for frozen commitments once the pending instance is gone. The check is defensive.

### `InstancePermanentlyDeletedEvent` → `_onCommitmentDeleted(event)`

Deletes `GarmentProfile` and all `CommitmentWeeklyProgress` records for this `definitionId`.

### `TemporalHelperService.onWeekEnded` → `_onWeekEnded(event)`

Seals the live `CommitmentWeeklyProgress` record — sets `isCurrentWeek: false`. Creates new live record for the coming week. Garment delta calculation happens live on `PerformanceUpdatedEvent`, not here.

---

## Internal Functions

### `_getPerformance(definitionId, livePerformance)` → double

```
if AppConfig.garmentUsesAccelerator == false:
  return livePerformance
return AcceleratorService.getMultiplier(definitionId) * livePerformance
```

### `_detectAchievements(profile, previousPercent)`

Central achievement detection for the Garment feature. Called after every garment update. Checks all garment achievement conditions and calls `_addAchievement()` for each that is met. Adding a new garment achievement (e.g. fortify phase completion) means adding one check here — nothing else changes.

```
// completion detection — fires only on transition from incomplete to complete
isComplete = profile.completionPercent >= 100.0 (Do) or <= 0.0 (Avoid)
wasComplete = previousPercent >= 100.0 (Do) or <= 0.0 (Avoid)

if isComplete and not wasComplete:
  _addAchievement(AchievementSubtype.garmentCompleted, profile)
```

### `_addAchievement(subtype, profile)`

Private. Constructs and submits an `AchievementRecord`.

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

No separate model, no repository write in Garment — the `AchievementRecord` is stored entirely in the Achievements feature.

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

Used by the commitment detail screen.

### `getWeeklyProgress(definitionId, limit?)` → List<CommitmentWeeklyProgress>

Current week first. Used by the weekly progress table.

### `getLiveWeekDelta(definitionId)` → double

Running delta for the current week.

---

## Rules

- The only writer of `GarmentProfile` and `CommitmentWeeklyProgress`
- Reads definition once at garment creation — for `GarmentTypeResolver` input only
- All four resolver/calculator functions are injected — never instantiated inside the service
- Garment updates are idempotent — `lastUpdatedDate` prevents double-application
- `_detectAchievements` fires only on genuine transitions — idempotent by design
- `GarmentRenderer` used only by `component_garment_display` — never called here

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceCreatedEvent`, `InstancePermanentlyDeletedEvent`
- `PerformanceService` — subscribes to `PerformanceUpdatedEvent`; calls `isWindowSuccess()`
- `TemporalHelperService` — subscribes to `onWeekEnded`
- `AchievementService.addAchievement()` — records garment achievements
- `GarmentRepository` — reads and writes `GarmentProfile` and `CommitmentWeeklyProgress`
- `CommitmentService.getDefinition()` — reads definition once at garment creation
- `AcceleratorService.getMultiplier()` — via `_getPerformance()`. `AcceleratorService` calls `StreakService.getStreakRecord()` directly — Streak is below Garment, a valid downward call
- `GarmentTypeResolver`, `ThreadColorResolver`, `GarmentDeltaCalculator` — injected

---

## Later Improvements

**Fortify phase.** After a garment completes, it enters a fortify phase — the user keeps building and the garment gains a visual reinforcement layer (golden thread border). When the fortify phase completes, `_detectAchievements` detects the transition and calls `_addAchievement()` with a `garmentFortified` subtype. Requires a `GarmentPhase` enum field on `GarmentProfile` and a new `AchievementSubtype` enum value.

**Gold phase.** After fortify, a gold phase begins — full gold visual treatment representing mastery. Completing it calls `_addAchievement()` with `garmentGolded`. Each phase is a new milestone in the commitment's lifetime and earns its own point value. The `_detectAchievements` function handles all phase transitions in one place — adding a new phase means one new condition check.

These phases make long-running commitments feel progressively rewarding rather than plateauing at completion.