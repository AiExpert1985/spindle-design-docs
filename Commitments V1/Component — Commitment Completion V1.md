
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 1

**Standalone:** yes — can be added or removed without touching other components.

**Purpose:** lets users mark a commitment as done — permanently finished, not paused. Completed commitments are achievements. History and streaks are preserved forever.

---

## Completed vs Frozen

||Frozen|Completed|
|---|---|---|
|Intent|Temporary pause, will resume|Done — ran its course|
|Reversible|Yes|No|
|Icon|❄|✓|
|History|Preserved|Preserved permanently|
|Streaks|Paused|Preserved permanently|
|New instances|None while frozen|None ever again|

---

## Why No Undo

Completion is a deliberate achievement moment — "I built this habit, it's part of me now." Making it reversible cheapens it. If a user wants to restart a completed commitment, they create a new one. The history of the original remains intact and readable.

---

## UI

Accessed via the ⋮ menu on any commitment card.

**Confirmation sheet:**

```
Mark "Morning Walk" as complete?

You're done with this commitment.
Your history and streaks are preserved forever.

[ Mark Complete ]   [ Cancel ]
```

On confirm → brief celebration animation → commitment moves to Completed section.

**Completed commitment:**

- Hidden from dashboard and threads screen by default
- Visible via the Completed filter
- All history, logs, and streaks remain readable
- No logging possible — read only

---

## Service Logic

- `CommitmentService.completeCommitment(id)`
    - Sets `isCompleted: true` and `completedAt: now` on definition
    - Stops instance generation
    - Preserves all history, log entries, and streak data untouched

---

## Dependencies

- `CommitmentService` — completion operation
- Presentation layer — brief celebration animation on confirm, then moves card to Completed section