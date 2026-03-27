**File Name**: model_accelerator_record **Feature**: Garment **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the current acceleration multiplier for one commitment's garment. One record per commitment. The multiplier modifies how fast the garment grows or dissolves — it never affects raw performance scores, cups, or any feature other than Garment.

---

## What the Accelerator Is

The accelerator is a single multiplier per commitment. It starts at 1.0 (neutral — no effect on garment speed). It is adjusted on each garment update based on the commitment's current streak: positive streaks push the multiplier toward the ceiling, negative streaks push it toward the floor. The garment delta calculation multiplies the base contribution by this value — so a user building consistent momentum grows their garment faster, and a user in a sustained struggle dissolves an avoid-habit garment more slowly.

The accelerator is internal to the Garment feature. No other feature reads or writes it. It is a Garment concern only.

---

## Fields

```
AcceleratorRecord
  definitionId: String     // the commitment this accelerator belongs to
                           // also the document ID in storage
  multiplierValue: double  // current multiplier — starts at 1.0
                           // bounded by AppConfig.acceleratorFloor and acceleratorCeiling
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