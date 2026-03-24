
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 3

**Standalone:** yes — entirely independent, uses existing service functions.

**Purpose:** Android home screen widget showing Today Score and quick-log buttons per commitment. Reduces logging friction to one tap without opening the app.

---

## What It Shows

```
Spindle  83%

Morning Walk  ████░  60%  [ +1 ]
No alcohol    ████████ 100% ✓
Read daily    ░░░░░░   0%  [ ✓ ]
```

- Today Score at top
- One row per active commitment: name, progress bar, quick-log button
- Quick-log button uses `defaultLogIncrement` from the commitment definition

---

## Behavior

- Tap quick-log button → logs `defaultLogIncrement`, widget updates immediately
- Tap anywhere else → opens app on dashboard
- Updates every time the app syncs — not real-time

---

## Service Logic

- Reads `CommitmentService.getTodayInstances()` for progress data
- Quick-log tap → `CommitmentService.logEntry(instanceId, defaultLogIncrement)`
- Widget refresh handled by WorkManager periodic task

---

## Dependencies

- `CommitmentService` — instance data and logging
- WorkManager — periodic widget refresh
- All tiers — widget is free