
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 2

**Standalone:** yes — plugs into existing code with zero changes to any Phase 1 service or model.

**Purpose:** automatically freezes and unfreezes a commitment on selected days of the week. Removes the need for manual freeze/unfreeze on predictable rest days.

---

## Why This Is Useful

Manual freeze requires the user to remember to freeze before a rest day and unfreeze after. For predictable patterns — weekends, Friday rest days, Ramadan — this is impractical. Scheduled freeze handles it automatically.

---

## How It Works

The process is simple and fully contained in this component:

**Setup (user action):** User selects which days of the week to auto-freeze for a commitment. This updates `freezeScheduleDays` on the commitment definition via the existing `CommitmentService.updateCommitment()` — no new service function needed.

**Midnight trigger (background scheduler):** Every midnight, a background scheduler trigger reads all active commitment definitions. For each commitment with `freezeScheduleDays` set:

- If today's day is in `freezeScheduleDays` AND commitment is not frozen → call `CommitmentService.freezeCommitment(id)`
- If today's day is NOT in `freezeScheduleDays` AND commitment is frozen due to schedule → call `CommitmentService.unfreezeCommitment(id)`

That's the entire component. It calls existing service functions — it adds no new ones.

---

## What Changes in Phase 2

|What|Change|
|---|---|
|CommitmentDefinition model|None — `freezeScheduleDays` already exists, just null in Phase 1|
|CommitmentService|None — uses existing `freezeCommitment()` and `unfreezeCommitment()`|
|CommitmentRepository|None — `saveDefinition` already saves all fields|
|BackgroundScheduler|Add one new midnight trigger|
|UI|Add day-picker to commitment form/settings|

Zero changes to existing Phase 1 code. This component is purely additive.

---

## UI

Accessed from the commitment form or ⋮ menu → "Schedule freeze days"

```
Auto-freeze on:
[ Mon ] [ Tue ] [ Wed ] [ Thu ] [ Fri ] [SAT✓] [SUN✓]

Commitment will freeze automatically on selected days
and resume the following day.

[ Save ]
```

Active schedule shown on commitment card with 📅 icon alongside ❄ when frozen.

---

## Data

No new model or repository needed. The schedule lives on `CommitmentDefinition.freezeScheduleDays` — a nullable list of integers (1–7 representing Mon–Sun). When null, no schedule is active.

Deleting a commitment automatically removes its schedule — no cleanup needed.

---

## Dependencies

- `CommitmentService.freezeCommitment()` — called by midnight trigger
- `CommitmentService.unfreezeCommitment()` — called by midnight trigger
- `CommitmentService.updateCommitment()` — called when user saves schedule
- `BackgroundScheduler` — midnight trigger registration