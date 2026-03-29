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
- **completionPercent** — accumulated weaving progress. Do: starts at 0.0, increases with each successful day. Avoid: starts at 100.0, decreases toward 0.0. No upper bound — exceeding 100% means the habit is complete and the user has entered fortify cycles (iron at 200%, gold at 300%, diamond at 400%). Lower bound is 0.0. Updated via subtract-add on every day record recalculation.
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