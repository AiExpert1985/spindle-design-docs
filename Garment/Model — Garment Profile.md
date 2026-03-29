**File Name**: model_garment_profile **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** the garment record for one commitment. Owns the visual identity (type and colors) and the running completion total. Entirely independent of `CommitmentDefinition` — linked only by `definitionId`. No garment fields ever appear on `CommitmentDefinition`.

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
  createdAt: DateTime
  updatedAt: DateTime
```

- **id** — unique identifier. Immutable.
- **definitionId** — the commitment this garment belongs to. Immutable.
- **commitmentType** — Do or Avoid. Determines fill direction. Immutable after creation.
- **garmentType** — the garment shape assigned at creation. Immutable — part of the commitment's visual identity. See `service_garment_type_resolver`.
- **threadColors** — hex color list assigned once at creation. Immutable. See `service_thread_color_resolver`.
- **completionPercent** — running total of all day record deltas. Do: 0.0 (empty) → 100.0 (complete). Avoid: 100.0 (full grip) → 0.0 (unraveled). Updated via subtract-add on every day record recalculation — never derived by summing all records. Clamped to 0.0–100.0.
- **createdAt** — immutable.
- **updatedAt** — updated on every write.

---

## Supporting Types

```
enum GarmentType {
  threadBracelet
  sock
  glove
  scarf
  sweater
}
```

---

## Rules

- Created by `GarmentService` on `InstanceCreatedEvent` for the first instance of a new commitment
- `garmentType` and `threadColors` assigned once at creation — never changed
- `completionPercent` updated only by `GarmentService` via subtract-add — no other writer
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`