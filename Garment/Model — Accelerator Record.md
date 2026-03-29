**File Name**: model_accelerator_record **Feature**: Garment **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** stores the current acceleration multiplier for one commitment's garment. One record per commitment. Internal to the Garment feature — no other feature reads or writes it.

---

## Fields

```
AcceleratorRecord
  definitionId: String     
  multiplierValue: double
  createdAt: DateTime
  updatedAt: DateTime
```

- **definitionId** — immutable. Links this record to one commitment.
- **multiplierValue** — the only value `GarmentService` reads. Starts at 1.0. Bounded at floor and ceiling defined in `AppConfig`. Never goes outside those bounds.

---

## Rules

- One record per commitment — created lazily by `AcceleratorService._getOrCreateRecord()` on first `getMultiplier()` call for a given commitment
- Written only by `AcceleratorService`
- Read only by `AcceleratorService`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`
- `CommitmentDefinition` has no accelerator fields — all accelerator state lives here