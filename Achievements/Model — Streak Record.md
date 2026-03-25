**File Name**: model_streak_record **Feature**: Achievements (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** stores the current streak state for one commitment. One record per commitment. No history — the current signed value and the all-time best are all that is needed.

---

## What a Streak Is

A streak is a single signed integer per commitment. Positive means consecutive kept windows. Negative means consecutive missed windows. Zero means neutral — either the commitment just started, or a streak just broke.

**Positive streak** — the user has kept this commitment for consecutive days. The longer the streak, the stronger the momentum signal.

**Negative streak** — the user has missed this commitment for consecutive days. The longer the negative streak, the stronger the struggling signal.

**Zero** — neutral state. A positive streak that breaks goes to zero first before going negative on the next miss. This is intentional — one missed day after a long kept streak is a neutral moment, not immediately a bad streak. The pattern of missing is what matters.

---

## Transition Rules

```
Current positive, day kept   → increment by 1   (e.g. +5 → +6)
Current positive, day missed → reset to 0        (e.g. +5 → 0)
Current zero,     day kept   → increment to +1
Current zero,     day missed → decrement to -1
Current negative, day kept   → reset to 0        (e.g. -3 → 0)
Current negative, day missed → decrement by 1    (e.g. -3 → -4)
```

Frozen windows are neutral — neither increment nor decrement. The streak is preserved through a freeze period unchanged.

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

- **definitionId** — the commitment this record belongs to. Also the document ID in storage — no separate `id` field needed.
- **currentStreak** — the live signed streak value. Updated on every window close.
- **bestStreak** — tracks only positive peaks. Updated when `currentStreak > bestStreak`. Never affected by negative streaks.
- **createdAt** — set once on first window close for this commitment.
- **updatedAt** — updated on every write.

---

## Rules

- One record per commitment — created on first window close, never recreated
- `CommitmentDefinition` has no streak fields — all streak state lives here
- Frozen windows leave the record unchanged
- `bestStreak` updated only when `currentStreak` is positive and exceeds the current best
- Written only by `StreakService`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`