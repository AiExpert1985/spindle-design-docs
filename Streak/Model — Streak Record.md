**File Name**: model_streak_record **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** stores the current streak state for one commitment. One record per commitment. Holds the signed current streak and the all-time best positive value.

---

## Fields

```
StreakRecord
  definitionId: String
  currentStreak: int     // signed — positive good, negative bad, zero neutral
  bestStreak: int        // all-time peak positive value — never decreases
  createdAt: DateTime
  updatedAt: DateTime
```

- **definitionId** — the commitment this record belongs to. Also the document ID in storage — no separate `id` field needed. One record per commitment, `definitionId` is the unique key.
- **currentStreak** — the live signed streak value. Updated on every window evaluation.
- **bestStreak** — tracks only positive peaks. Updated when `currentStreak > bestStreak`. Never affected by negative streaks.
- **createdAt** — set once on first window evaluation for this commitment.
- **updatedAt** — updated on every write.

---

## Rules

- One record per commitment — created on first window evaluation, never recreated
- `CommitmentDefinition` has no streak fields — all streak state lives here
- Frozen windows leave the record unchanged
- `bestStreak` updated only when `currentStreak` is positive and exceeds the current best
- Written only by `StreakService`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`