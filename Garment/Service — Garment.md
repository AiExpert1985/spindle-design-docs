**File Name**: service_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** owns the complete lifecycle of garment profiles. Initializes garments when commitments are created, updates them live as performance changes, and exposes read functions to the presentation layer. The single writer of `GarmentProfile` and `CommitmentWeeklyProgress`.

Fully independent — subscribes to events from below, writes nothing outside its own models, and can be removed without affecting any other feature.

---

## Abstractions

```
GarmentTypeResolver (abstract)
  → RuleBasedGarmentTypeResolver      // deterministic rules based on commitment shape
  → AIGarmentTypeResolver             // AI-assisted, Pro/Premium only (later)

ThreadColorResolver (abstract)
  → RandomVividColorResolver          // curated random palette from vivid color set
  → UserChosenColorResolver           // user picks colors (later)

GarmentDeltaCalculator (abstract)
  → DefaultGarmentDeltaCalculator     // rule-based: contribution + decay + streak bonus

GarmentRenderer (abstract)
  → SimpleThreadRenderer              // flat thread fill, visible strands
```

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onCommitmentCreated(event)`

Creates a `GarmentProfile` for this commitment:

1. `GarmentTypeResolver.resolve(definition)` → `garmentType`
2. `ThreadColorResolver.resolve(definitionId)` → `threadColors`
3. Creates profile with `completionPercent` at starting value — Do: 0.0%, Avoid: 100.0%

Idempotent — exits silently if profile already exists.

### `PerformanceUpdatedEvent` → `_onPerformanceUpdated(event)`

```
1. Read current GarmentProfile
2. Check lastUpdatedDate — if already updated today, exit (idempotent)
3. if snapshot.commitmentState == frozen: exit  // safety net
4. performanceValue = _getPerformance(event.definitionId, event.livePerformance)
5. isSuccess = PerformanceService.isWindowSuccess(event.livePerformance)
6. delta = GarmentDeltaCalculator.calculate(profile, performanceValue, isSuccess)
7. previousPercent = profile.completionPercent
8. Apply delta — clamp to 0.0–100.0
9. Save updated profile
10. Update live CommitmentWeeklyProgress record
11. Publish GarmentUpdatedEvent
12. Check for completion transition — call _onGarmentCompleted if newly complete
```

**Why the frozen check is a safety net:** `PerformanceUpdatedEvent` only fires from `InstanceCreatedEvent` and `ActivityEvent` — neither fires for frozen commitments once the pending instance is gone. The check is defensive.

**Completion detection (step 12):**

```
isComplete = newPercent >= 100.0 (Do) or newPercent <= 0.0 (Avoid)
wasComplete = previousPercent >= 100.0 (Do) or previousPercent <= 0.0 (Avoid)

if isComplete and not wasComplete:
  _onGarmentCompleted(profile)
```

Only fires on transition from incomplete to complete — never re-fires.

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

### `_onGarmentCompleted(profile)`

Called when garment transitions from incomplete to complete.

```
AchievementService.addAchievement(AchievementRecord(
  type: AchievementType.garment,
  subtype: AchievementSubtype.garmentCompleted,
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

### `getWeeklyProgress(definitionId, limit?)` → List < CommitmentWeeklyProgress > 

Current week first. Used by the weekly progress table.

### `getLiveWeekDelta(definitionId)` → double

Running delta for the current week.

---

## Rules

- The only writer of `GarmentProfile` and `CommitmentWeeklyProgress`
- Reads definition once at garment creation — for `GarmentTypeResolver` input only
- All four resolver/calculator functions are injected
- Garment updates are idempotent — `lastUpdatedDate` prevents double-application
- `_onGarmentCompleted` fires only on transition — never re-fires for already-complete garments
- `GarmentRenderer` used only by `component_garment_display` — never called here

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceCreatedEvent`, `InstancePermanentlyDeletedEvent`
- `PerformanceService` — subscribes to `PerformanceUpdatedEvent`; calls `isWindowSuccess()`
- `TemporalHelperService` — subscribes to `onWeekEnded`
- `AchievementService.addAchievement()` — records garment completion achievement
- `GarmentRepository` — reads and writes `GarmentProfile` and `CommitmentWeeklyProgress`
- `CommitmentService.getDefinition()` — reads definition once at garment creation
- `AcceleratorService.getMultiplier()` — via `_getPerformance()`. `AcceleratorService` calls `StreakService.getStreakRecord()` directly — Streak is below Garment, a valid downward call
- `GarmentTypeResolver`, `ThreadColorResolver`, `GarmentDeltaCalculator` — injected

---

## Later Improvements

**Fortify phase.** After completion, the garment enters a fortify phase — the user keeps building and the garment gains a visual reinforcement layer (golden thread border). Completing fortify calls `AchievementService.addAchievement()` with a `garmentFortified` subtype. Requires a `GarmentPhase` field on `GarmentProfile`.

**Gold phase.** After fortify, a gold phase begins — full gold visual treatment representing mastery. Completing it records `garmentGolded`. Each phase produces its own achievement and point value, making long-running commitments feel progressively rewarding rather than plateauing at completion.