**File Name**: model_garment_day_record **Feature**: Garment **Phase**: 3 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** stores the garment contribution for one commitment on one day. One record per commitment per day. The delta starts at zero and is recalculated whenever performance changes for that day. `GarmentProfile.completionPercent` is the running sum of all deltas.

---

## Fields

```
GarmentDayRecord
  id: String
  definitionId: String
  date: Date
  accelerationValue: double
  delta: double
  createdAt: DateTime
  updatedAt: DateTime
```

- **id** — client-generated. Immutable.
- **definitionId** — the commitment this record belongs to.
- **date** — the calendar date this record covers. Combined with `definitionId` forms the unique key.
- **accelerationValue** — the accelerator multiplier snapshotted at record creation (when the day's instance opens). Immutable after creation. Reflects streak momentum going _into_ this day, not during it. `GarmentService` reads this value and multiplies it with `livePerformance` to produce `performanceValue` before calling the delta calculator — ensuring all recalculations for this day use the same acceleration regardless of when they fire.
- **delta** — how much the garment moved on this day. Starts at 0.0 when the record is created. Recalculated on every `PerformanceUpdatedEvent` for this day's instance: `delta = (performanceValue / 100) × dailyFullContribution`, where `performanceValue` already incorporates the acceleration multiplier. Missed days (zero performance) produce zero delta — nothing is ever subtracted from a day's contribution for missed days.
- **createdAt** — immutable.
- **updatedAt** — updated on every delta recalculation.

---

## Rules

- One record per commitment per day — `definitionId + date` is the unique key
- Created when the day's instance is created — not lazily on first performance event
- `accelerationValue` is immutable after creation — never updated even if the accelerator changes during the day
- `delta` recalculated on every performance change for this day using the stored `accelerationValue`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`
- To reconstruct garment state at any historical date: sum all `delta` values up to and including that date