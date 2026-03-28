**File Name**: service_activity **Feature**: Activity **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** records, edits, and deletes user activity log entries. Validates the backfill window before every write. Publishes one event type for all three operations. Calls no other service after writing — all downstream reactions happen through subscriptions.

---

## Events Published

```
ActivityEvent
  type: ActivityEventType   // created | updated | deleted
  definitionId: String
  loggedAt: DateTime
  value: double?            // null on deleted
```

Published after every successful write, edit, or delete.

---

## Write Functions

All write functions return `Result<void>`. A `Failure` is returned — never a silent rejection — when a validation rule is violated. The presentation layer receives the failure and shows the user an appropriate message.

### `recordEntry(definitionId, value, loggedAt, note?) → Result<void>`

Fails if `loggedAt` is outside the backfill window (more than `AppConfig.maxLogBackfillDays` days ago, or in the future). The UI shows the user that the date is outside the allowed range.

On success:

1. Write `LogEntry(id, definitionId, value, loggedAt, note, createdAt: now, updatedAt: now)`
2. Publish `ActivityEvent(type: created, definitionId, loggedAt, value)`

For binary commitments the caller passes `value: 1`. No special path.

### `editEntry(entryId, value, note?) → Result<void>`

Fails if the entry is not found, or if `entry.loggedAt` is outside the backfill window. The UI shows the user that the entry can no longer be edited.

On success:

1. Update `value`, `note`, `updatedAt: now`
2. Publish `ActivityEvent(type: updated, definitionId, loggedAt, value)`

Only `value` and `note` are editable. `loggedAt` is immutable.

### `deleteEntry(entryId) → Result<void>`

Fails if the entry is not found, or if `entry.loggedAt` is outside the backfill window. The UI shows the user that the entry can no longer be deleted.

On success:

1. Delete entry
2. Publish `ActivityEvent(type: deleted, definitionId, loggedAt, value: null)`

---

## Read Functions

### `getTotalLoggedForCommitmentOnDate(definitionId, date) → double`

Fetches all entries for `definitionId` on `date` via `ActivityRepository.getEntriesForDay()`, then sums their `value` fields in Dart. Returns 0.0 if no entries exist.

### `getEntriesForDay(definitionId, date) → List<LogEntry>`

All log entries for a specific day ordered by `loggedAt` descending.

### `getEntriesForWeek(definitionId, weekStart) → List<LogEntry>`

All log entries for the Mon–Sun week containing `weekStart`.

### `getEntriesForPeriod(definitionId, from, to) → List<LogEntry>`

All log entries in an arbitrary date range.

---

## Event Subscriptions

### `InstancePermanentlyDeletedEvent` (CommitmentIdentityService)

Deletes all log entries for `event.definitionId`. `ActivityService` owns its own data and cleans it up when the commitment is permanently removed.

---

## Dependencies

- `ActivityRepository` — log entry storage
- `CommitmentIdentityService` — subscribes to `InstancePermanentlyDeletedEvent`
- `AppConfig` — `maxLogBackfillDays`

---

## Rules

- All write functions return `Result<void>` — failures are never silent
- `loggedAt` always passed by caller in `recordEntry` — never defaulted to `now` internally
- `loggedAt` immutable after creation — `editEntry` never changes it
- Backfill window enforced on all three write functions — out-of-window attempts return `Failure`
- All aggregations computed in Dart after fetching raw records — never delegated to the repository