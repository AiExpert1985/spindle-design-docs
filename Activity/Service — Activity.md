**File Name**: service_activity **Feature**: Activity **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** records, edits, and deletes user activity log entries. Publishes one event type for all three operations. All downstream reactions happen through subscriptions — this service calls nothing after writing.

---

## Independence

`ActivityService` is almost entirely self-contained. It receives requests from the user, validates the date, writes the entry, and publishes one event. It calls no other service and checks no state from other features.

The one exception: `ActivityService` subscribes to `InstancePermanentlyDeletedEvent` from `CommitmentService` to delete its own log entries when a commitment is permanently removed. `ActivityService` owns its own data and cleans it up when the parent commitment is gone.

Logging is allowed regardless of commitment state — frozen, completed, or active. The backfill window is the only gate.

---

## Events Published

```
ActivityEvent
  type: ActivityEventType   // created | updated | deleted
  definitionId: String
  loggedAt: DateTime
  value: double?            // null on deleted
```

Published after every successful write, edit, or delete. `PerformanceService` subscribes and recalculates `livePerformance` for the matching instance on every type. `Encouragement` subscribes to `created` only for immediate log feedback.

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

Fetches all entries for `definitionId` on `date` via `ActivityRepository.getEntriesForDay()`, then sums their `value` fields in Dart. Returns 0.0 if no entries exist. Called by `PerformanceService` after each recalculation.

### `getEntriesForDay(definitionId, date) → List<LogEntry>`

All log entries for a specific day ordered by `loggedAt` descending. Used by `LogHistorySheet`.

### `getEntriesForWeek(definitionId, weekStart) → List<LogEntry>`

All log entries for the Mon–Sun week containing `weekStart`. Used by `LogHistorySheet` week view.

### `getEntriesForPeriod(definitionId, from, to) → List<LogEntry>`

All log entries in an arbitrary date range. Used by Analytics.

---

## Event Subscriptions

### `InstancePermanentlyDeletedEvent` (Commitment)

Deletes all log entries for `event.definitionId`. `ActivityService` owns its own data and cleans it up when the commitment is permanently removed.

---

## Dependencies

- `ActivityRepository` — log entry storage
- `CommitmentService` — subscribes to `InstancePermanentlyDeletedEvent`
- `AppConfig` — `maxLogBackfillDays`

---

## Rules

- All write functions return `Result<void>` — failures are never silent
- `loggedAt` always passed by caller in `recordEntry` — never defaulted to `now` internally
- `loggedAt` immutable after creation — `editEntry` never changes it
- Backfill window enforced on all three write functions — out-of-window attempts return `Failure`
- No commitment state check — logging allowed regardless of commitment state
- No calls to any other service after writing — all downstream reactions through `ActivityEvent`
- All aggregations computed in Dart after fetching raw records — never delegated to the repository