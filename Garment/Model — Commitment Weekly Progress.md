**File Name**: model_commitment_weekly_progress **Feature**: Garment **Phase**: 3 **Created**: 18-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** stores the weekly garment progress delta for one commitment. One record per commitment per week. Used by the commitment detail screen's weekly progress table to show how the garment grew or decayed each week.

---

## What It Represents

Each week, the garment moves by some net amount — the sum of daily contributions, decay penalties, and streak bonuses over that Mon–Sun period. `CommitmentWeeklyProgress` captures that net movement as a single delta value, making the weekly history readable without re-calculating from raw instances.

---

## Fields

```
CommitmentWeeklyProgress
  id: String
  definitionId: String
  weekStart: DateTime        // Monday 00:00:00 of this week
  weeklyDelta: double        // net change in completionPercent this week
                             // positive = progress
                             // negative = decay exceeded progress
  isCurrentWeek: bool        // true for the live in-progress record
                             // false once sealed on Sunday night
  createdAt: DateTime
  updatedAt: DateTime        // updated daily during the live week
```

- **weekStart** — always a Monday. Combined with `definitionId` forms the unique key.
- **weeklyDelta** — net garment movement for the week. Stored at calculation time, never re-derived. Naturally includes all forces — daily contributions, decay, streak bonus — because it is simply `completionPercent` at week end minus `completionPercent` at week start.
- **isCurrentWeek** — the live record is updated daily as the week progresses. Sealed to false on Sunday night after the final daily update.

---

## Two Record States

**Live record** — created on Monday when the week begins. `isCurrentWeek: true`. Updated daily by `GarmentCalculationService` as the garment moves. Converted to sealed on Sunday night.

**Sealed record** — `isCurrentWeek: false`. Immutable after sealing. Represents the final delta for that Mon–Sun period.

---

## Rules

- One record per commitment per week — `definitionId + weekStart` is the unique key
- `weeklyDelta` is stored, never re-derived
- Sealed records are immutable — never updated after `isCurrentWeek: false`
- Frozen weeks produce no record — frozen commitments skip the daily garment update
- Records are deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`
- Historical records are preserved on commitment completion — history is never erased on completion
