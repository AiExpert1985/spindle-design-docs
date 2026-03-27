**File Name**: service_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** owns the complete lifecycle of garment profiles. Initializes garments when commitments are created, updates them live as performance changes, and exposes read functions to the presentation layer. The single writer of `GarmentProfile` and `CommitmentWeeklyProgress`.

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

### `PerformanceUpdatedEvent` → `_onPerformanceUpdated(event)`

Fires on every `livePerformance` change. Triggers a live garment update using the current performance value.

```
1. Read current GarmentProfile
2. Check lastUpdatedDate — if already updated today, exit (idempotent)
3. if snapshot.commitmentState == frozen: exit  // safety net — see note below
4. performanceValue = _getPerformance(event.definitionId, event.livePerformance)
5. isSuccess = PerformanceService.isWindowSuccess(event.livePerformance)
6. delta = GarmentDeltaCalculator.calculate(profile, performanceValue, isSuccess)
7. Apply delta to completionPercent — clamp to 0.0–100.0
8. Save updated profile via GarmentRepository
9. Update live CommitmentWeeklyProgress record
10. Publish GarmentUpdatedEvent
```

**Why the frozen check is a safety net, not the primary gate:** when a commitment is frozen, the pending instance is cleared but the instance still closes unconditionally on the Heartbeat tick. `PerformanceUpdatedEvent` fires on `InstanceCreatedEvent` and `ActivityEvent` — neither of which fire for frozen commitments once the pending instance is gone. The frozen check exists as a defensive guard in case a stray performance event arrives for a frozen commitment. It is not the primary mechanism.

### `InstancePermanentlyDeletedEvent` → `_onCommitmentDeleted(event)`

Deletes the `GarmentProfile` and all `CommitmentWeeklyProgress` records for this `definitionId`.

### `WeekEndedEvent` → `_onWeekEnded(event)`

Seals the live `CommitmentWeeklyProgress` record for all commitments — sets `isCurrentWeek: false` so the record becomes an immutable historical fact. Creates a new live record for the coming week.

This is the only reason Garment subscribes to `WeekEndedEvent` — the garment delta calculation itself happens live on `PerformanceUpdatedEvent`, not here.

---

## Internal Functions

### `_getPerformance(definitionId, livePerformance)` → double

The single point where Garment decides which performance value to use. Abstracts the source completely — `GarmentDeltaCalculator` receives one number and never knows where it came from.

```
if AppConfig.garmentUsesAccelerator == false:
  return livePerformance

return AcceleratorService.getMultiplier(definitionId) * livePerformance
```

When the accelerator is disabled, this is a pass-through — zero overhead, zero coupling.

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
- Never reads `CommitmentDefinition` directly for ongoing updates — all needed data comes from events
- Reads definition once at garment creation — for `GarmentTypeResolver` input only
- All four resolver/calculator functions are injected — never instantiated inside the service
- Garment updates are idempotent — `lastUpdatedDate` prevents double-application on the same day
- Frozen commitments skipped — defensive check in `_onPerformanceUpdated`
- `GarmentRenderer` is used only by `component_garment_display` — never called by this service

---

## Dependencies

- `CommitmentService` — subscribes to `InstanceCreatedEvent`, `InstancePermanentlyDeletedEvent`, `WeekEndedEvent`
- `PerformanceService` — subscribes to `PerformanceUpdatedEvent`; calls `isWindowSuccess()`
- `GarmentRepository` — reads and writes `GarmentProfile` and `CommitmentWeeklyProgress`
- `CommitmentService.getDefinition()` — reads definition once at garment creation
- `AcceleratorService.getMultiplier()` — called by `_getPerformance()` when `garmentUsesAccelerator` is true. `AcceleratorService` calls `StreakService.getStreakRecord()` directly — Streak is below Garment in the chain, a valid downward call
- `GarmentTypeResolver`, `ThreadColorResolver`, `GarmentDeltaCalculator` — injected