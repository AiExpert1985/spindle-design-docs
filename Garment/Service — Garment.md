**File Name**: service_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** owns the complete lifecycle of garment profiles. Initializes garments when commitments are created, updates them daily as performance accumulates, and exposes read functions to the presentation layer. The single writer of `GarmentProfile` and `CommitmentWeeklyProgress`.

Fully independent — subscribes to events from below, writes nothing outside its own models, and can be removed without affecting any other feature.

---

## Abstractions

The service delegates to four pure, replaceable functions. Each is an abstract interface with one concrete implementation in Phase 3. Replacing any implementation requires no changes to the service or any caller.

```
GarmentTypeResolver (abstract)
  → RuleBasedGarmentTypeResolver      // deterministic rules based on commitment shape
  → AIGarmentTypeResolver             // AI-assisted, Pro/Premium only (later)

ThreadColorResolver (abstract)
  → RandomVividColorResolver          // curated random palette from vivid color set
  → UserChosenColorResolver           // user picks colors (later)

GarmentDeltaCalculator (abstract)
  → DefaultGarmentDeltaCalculator     // rule-based: contribution + decay + streak bonus
  → (future: science-backed or AI-tuned implementation)

GarmentRenderer (abstract)
  → SimpleThreadRenderer              // flat thread fill, visible strands
  → (future: detailed woven texture, completion glow, highlighted new threads)
```

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onCommitmentCreated(event)`

Fires when the first instance of a new commitment is created. Creates a `GarmentProfile` for this commitment by calling:

1. `GarmentTypeResolver.resolve(definition)` → `garmentType`
2. `ThreadColorResolver.resolve(definitionId)` → `threadColors`
3. Creates `GarmentProfile` with `completionPercent` at starting value:
    - Do commitment → 0.0%
    - Avoid commitment → 100.0%

Idempotent — exits silently if a profile already exists for this `definitionId`.

Also fires when an existing commitment's instance is recreated after a definition change — the idempotency check prevents duplicate profile creation.

### `PerformanceUpdatedEvent` where `isClosed: true` → `_onWindowClosed(event)`

Fires when a commitment window closes with its final `livePerformance`. Triggers the daily garment update for this commitment.

```
1. Read current GarmentProfile
2. performanceValue = _getPerformance(event.definitionId, event.livePerformance)
3. delta = GarmentDeltaCalculator.calculate(profile, performanceValue, commitmentType)
4. Apply delta to completionPercent — clamp to 0.0–100.0
5. Save updated profile via GarmentRepository
6. Update live CommitmentWeeklyProgress record
7. Publish GarmentUpdatedEvent
```

Idempotent — checks `lastUpdatedDate` before running.

---

## Internal Functions

### `_getPerformance(definitionId, livePerformance)` → double

The single point where Garment decides which performance value to use. Abstracts the source completely — `GarmentDeltaCalculator` receives one number and never knows where it came from.

```
if AppConfig.garmentUsesAccelerator == false:
  return livePerformance

return AcceleratorService.getMultiplier(definitionId) * livePerformance
```

When the accelerator is disabled, this is a pass-through — zero overhead, zero coupling. When enabled, it multiplies the raw performance by the current multiplier for this commitment. The formula is entirely inside `AcceleratorService` — changing how the accelerator works requires no change here.

### `InstancePermanentlyDeletedEvent` → `_onCommitmentDeleted(event)`

Deletes the `GarmentProfile` and all `CommitmentWeeklyProgress` records for this `definitionId`.

### `WeekEndedEvent` → `_onWeekEnded(event)`

Seals the live `CommitmentWeeklyProgress` record for all commitments. Sets `isCurrentWeek: false`. Creates new live records for the coming week.

---

## Events Published

```
GarmentUpdatedEvent
  definitionId: String
  completionPercent: double
  weeklyDelta: double
```

Published after every garment update. Consumed by the presentation layer to update the garment display live.

---

## Read Functions

### `watchGarmentProfile(definitionId)`

Stream of the `GarmentProfile` for one commitment. Used by the commitment detail screen.

### `getWeeklyProgress(definitionId, limit?)`

Ordered list of `CommitmentWeeklyProgress` records, current week first. Used by the weekly progress table on the commitment detail screen.

### `getLiveWeekDelta(definitionId)`

The running delta for the current week. Used by the garment display to show "This week: +3% so far."

---

## Rules

- The only writer of `GarmentProfile` and `CommitmentWeeklyProgress`
- Never reads `CommitmentDefinition` directly for ongoing updates — all needed data comes from instance events
- Reads definition once at garment creation — for `GarmentTypeResolver` input only
- All four resolver/calculator functions are injected — never instantiated inside the service
- Frozen commitments are skipped — no garment update when `instance.commitmentState == frozen`
- `GarmentRenderer` is used only by `component_garment_display` — never called by this service

---

## Dependencies

- EventBus — subscribes to `InstanceCreatedEvent`, `PerformanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`, `WeekEndedEvent`; publishes `GarmentUpdatedEvent`
- `GarmentRepository` — reads and writes `GarmentProfile` and `CommitmentWeeklyProgress`
- `CommitmentService.getDefinition()` — reads definition once at garment creation
- `AcceleratorService.getMultiplier()` — called by `_getPerformance()` when `garmentUsesAccelerator` is true. `AcceleratorService` in turn calls `AchievementService.getStreakRecord()` — Garment never calls AchievementService directly
- `GarmentTypeResolver` — injected
- `ThreadColorResolver` — injected
- `GarmentDeltaCalculator` — injected
- `TemporalHelper` — day boundary and week start calculations