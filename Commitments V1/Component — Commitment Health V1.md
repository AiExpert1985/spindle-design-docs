
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 2
**Standalone:** yes — can be added or removed without touching other components.

**Purpose:** shows the current performance state of each active commitment as a colored dot. Gives the user an instant snapshot of which commitments are on track and which need attention.

---

## UI — Your Record

One colored dot per active commitment. Empty circles for available unused slots.

```
🟢  ⭐  🟠  🟢  🟢  ○  ○
5 active  ·  2 slots available
```

**Dot colors:**

|Color|Meaning|
|---|---|
|🟠 Orange|Below average — needs attention|
|🟢 Green|On track|
|⭐ Gold star|Excellent — exceeding target|
|○ Empty circle|Available slot, no commitment assigned|

Tap any commitment dot → navigates to Thread Detail screen for that commitment.

---

## What "Current Performance" Means

Each dot reflects the commitment's score for its current active window:

- A daily commitment shows today's progress
- A weekly commitment shows this week's progress so far
- A monthly commitment shows this month's progress so far

This means the row is always meaningful regardless of commitment type or recurrence.

---

## Score to Color Mapping

|Score|Color|
|---|---|
|< 40%|🟠 Orange|
|40–79%|🟢 Green|
|≥ 80%|⭐ Gold star|

---

## Service Logic

For each active commitment, fetch the current instance and read its `progressPercent`. Map to a color. No calculation stored — recomputed each time the row is rendered.

---

## Repository Operations

- Fetch active commitment definitions (read only)
- Fetch current instances for today (read only)

---

## Trigger

Rendered when Your Record screen opens. Updates in real time if the user logs a commitment while the screen is open (Riverpod watches instance state).

---

## Dependencies

- Commitment definitions and current instances (read only)
- Unlock engine component — the slot count (filled + empty circles) is shared with unlock engine UI. Both components read the same data independently.