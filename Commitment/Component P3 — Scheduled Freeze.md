**File Name**: component_scheduled_freeze **Feature**: Commitment **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** automatically freezes and unfreezes a commitment on a schedule — either on recurring days of the week or during a one-time date range. Removes the need for manual freeze on predictable patterns like rest days, travel, or Ramadan.

Fully standalone — plugs into existing services with no changes to any other feature. Purely additive.

---

## Two Freeze Modes

### Recurring Days

The user selects days of the week to auto-freeze, every week. Example: freeze on Friday and Saturday as rest days. The commitment freezes automatically at the start of each selected day and resumes the following morning.

Natural fit for: weekly rest days, religious observance days, recurring schedule gaps.

### One-Time Period

The user selects a start date and end date. The commitment freezes on the start date and unfreezes automatically on the day after the end date. Example: traveling Nov 10–17.

Natural fit for: travel, illness, Ramadan, holidays, any planned break with a known end date.

Both modes can be active simultaneously on the same commitment — a user can have recurring rest days configured and a one-time travel period scheduled at the same time.

---

## Model

```
FreezeSchedule
  id: String
  definitionId: String
  recurringDays: List<DayOfWeek>?     // null if no recurring schedule
  periodStart: DateTime?              // null if no one-time period
  periodEnd: DateTime?                // null if no one-time period
  createdAt: DateTime
  updatedAt: DateTime
```

Stored as a separate record linked to the commitment by `definitionId` — not added to `CommitmentDefinition`. This keeps the commitment model thin and allows the component to be removed cleanly.

---

## Service

`ScheduledFreezeService` subscribes to `LongIntervalTickEvent` at day boundary. On each day boundary, for every active commitment with a `FreezeSchedule`:

- **Recurring days:** if today is in `recurringDays` and commitment is active → `freezeCommitment(id)`. If today is not in `recurringDays` and commitment was frozen by this schedule → `unfreezeCommitment(id)`.
- **One-time period:** if today equals `periodStart` and commitment is active → `freezeCommitment(id)`. If today equals day after `periodEnd` and commitment was frozen by this schedule → `unfreezeCommitment(id)`.

The service tracks whether it triggered the freeze — it only calls `unfreezeCommitment()` for freezes it initiated, never for manual freezes the user set themselves.

All calls go through `CommitmentService` — the service never writes to `CommitmentDefinition` directly.

---

## UI

Accessed from two entry points:

- Commitment Form → later improvements section
- Commitment State Change component ⋮ menu → "Schedule Freeze"

**Recurring days picker:**

```
Auto-freeze on:
[ Mon ] [ Tue ] [ Wed ] [ Thu ] [ Fri ] [Sat✓] [Sun✓]

Commitment freezes automatically on selected days
and resumes the following day.

[ Save ]   [ Cancel ]
```

**One-time period picker:**

```
Freeze from:   [ 10 Nov 2026 ]
Freeze until:  [ 17 Nov 2026 ]

Commitment will freeze on Nov 10 and resume Nov 18.

[ Save ]   [ Cancel ]
```

Both pickers are shown in the same sheet, in separate sections. The user can configure one or both.

---

## Rules

- The service only unfreezes commitments it froze — never touches manually frozen commitments
- One-time period: end date must be after start date — validated before saving
- Recurring days: at least one day must be selected — empty selection is rejected
- Deleting a commitment deletes its `FreezeSchedule` automatically via `InstancePermanentlyDeletedEvent`
- No limit on how many commitments can have a schedule

---

## Dependencies

- EventBus — subscribes to `LongIntervalTickEvent` (day boundary), `InstancePermanentlyDeletedEvent` (cleanup)
- `CommitmentService.freezeCommitment()` — triggers freeze
- `CommitmentService.unfreezeCommitment()` — triggers unfreeze
- `TemporalHelper` — day boundary detection, day-of-week resolution
- `FreezeScheduleRepository` — stores and retrieves schedules