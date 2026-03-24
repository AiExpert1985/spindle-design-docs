**File Name**: model_garment_profile **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the garment record for one commitment. Owns everything the Garment feature knows about a commitment — type, colors, completion percent, and weekly progress. Entirely independent of `CommitmentDefinition` — linked only by `definitionId`.

No garment fields ever appear on `CommitmentDefinition`. The Garment feature is self-contained and removable without touching any other feature's model.

---

## What a Garment Profile Is

When a commitment is created, the Garment feature creates a corresponding `GarmentProfile`. The profile is the garment's persistent state — it accumulates weaving progress over the life of the commitment and carries the visual identity (type and colors) that makes each commitment's garment feel personal.

For Do commitments: the garment begins empty and fills toward complete as the habit builds.
For Avoid commitments: the garment begins complete and empties toward bare as the habit unravels.

---

## Fields

```
GarmentProfile
  id: String
  definitionId: String
  commitmentType: CommitmentType
  garmentType: GarmentType
  threadColors: List<String>
  completionPercent: double
  consecutiveFailDays: int
  lastUpdatedDate: Date?
  createdAt: DateTime
  updatedAt: DateTime
```

- **id** — unique identifier. Immutable.
- **definitionId** — the commitment this garment belongs to. Immutable.
- **commitmentType** — Do or Avoid. Determines fill direction. Immutable after creation.
- **garmentType** — the garment shape assigned at creation. Immutable — part of the commitment's visual identity. See `service_garment_type_resolver`.
- **threadColors** — hex color list assigned once at creation. Immutable. Used by the renderer to color individual threads consistently. See `service_thread_color_resolver`.
- **completionPercent** — current garment fill. Do: 0.0 (empty) → 100.0 (complete). Avoid: 100.0 (full grip) → 0.0 (unraveled). Updated by `GarmentCalculationService` once per day.
- **consecutiveFailDays** — how many consecutive days performance was below threshold. Used by `GarmentDeltaCalculator` for decay activation. Resets to 0 on any successful day.
- **lastUpdatedDate** — the last date the garment was updated. Used for idempotency — prevents double-application on the same day.
- **createdAt** — immutable.
- **updatedAt** — updated on every write.

---

## Supporting Types

```
enum GarmentType {
  thread_bracelet
  sock
  glove
  scarf
  sweater
}
```

---

## Rules

- Created by `GarmentService` on `InstanceCreatedEvent` for the first instance of a new commitment
- `garmentType` and `threadColors` are assigned once at creation — never changed
- `completionPercent` updated only by `GarmentCalculationService` — no other writer
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`
- `CommitmentDefinition` has no garment fields — this model owns all garment state
