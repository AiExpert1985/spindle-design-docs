**File Name**: model_streak_record **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** stores the current streak state for one commitment. One record per commitment. Holds the signed current streak and the all-time best positive value.

---

## Fields

```
StreakRecord
  definitionId: String
  currentStreak: int        // signed — positive good, negative bad, zero neutral
  bestStreakValue: int       // all-time peak positive value — never decreases
  bestStreakDate: DateTime?  // when bestStreakValue was last set — null until first positive streak
  createdAt: DateTime
  updatedAt: DateTime
```

- **definitionId** — the commitment this record belongs to. Also the document ID in storage. One record per commitment.
- **currentStreak** — the live signed streak value. Updated on every window evaluation.
- **bestStreakValue** — tracks only positive peaks. Updated when `currentStreak` exceeds `bestStreakValue`. Never affected by negative streaks.
- **bestStreakDate** — the date `bestStreakValue` was last set. Null until the first positive streak is achieved. Updated together with `bestStreakValue`.
- **createdAt** — set once on first window evaluation for this commitment.
- **updatedAt** — updated on every write.

---

## Rules

- One record per commitment — created on first window evaluation, never recreated
- `CommitmentDefinition` has no streak fields — all streak state lives here
- Frozen windows leave the record unchanged
- `bestStreakValue` and `bestStreakDate` updated together — only when `currentStreak` is positive and exceeds `bestStreakValue`
- Written only by `StreakService`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`