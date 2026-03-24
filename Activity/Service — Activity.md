**File Name**: activityservice **Feature**: Activity **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** records, edits, and deletes user activity log entries. Publishes one event type for all three operations. All downstream reactions happen through subscriptions — this service calls nothing after writing.

---

## Independence

ActivityService is almost entirely self-contained. It receives requests from the user, validates the date, writes the entry, and publishes one event. It calls no other service and checks no state from other features.

The one exception: ActivityService subscribes to `InstancePermanentlyDeletedEvent` to delete its own log entries when a commitment is permanently removed. ActivityService owns its own data and cleans it up when the parent commitment is gone.

Logging is allowed regardless of commitment state — frozen, completed, or active. The date restriction is the only gate.

---

## Events Published

```
ActivityEvent
  type: ActivityEventType   // created | updated | deleted
  definitionId: String
  loggedAt: DateTime
  value: double?            // null on deleted
```

Published after every write, edit, or delete. PerformanceService subscribes and recalculates `livePerformance` for the matching instance regardless of type. Encouragement subscribes to `created` only for immediate log feedback.

---

## Write Functions

### `recordEntry(definitionId, value, loggedAt, note?, contextTags?)`

1. Check `loggedAt` within backfill window — if not, return silently
2. Write `LogEntry(definitionId, value, loggedAt, note, contextTags)`
3. Publish `ActivityEvent(type: created, definitionId, loggedAt, value)`

For binary commitments, caller passes `value: 1`. No special path.

### `editEntry(entryId, value, note?)`

1. Read entry — if not found, return silently
2. Check entry `loggedAt` within backfill window — if not, return silently
3. Update `value`, `note`, `updatedAt: now`
4. Publish `ActivityEvent(type: updated, definitionId, loggedAt, value)`

Only `value` and `note` are editable. `loggedAt` is immutable.

### `deleteEntry(entryId)`

1. Read entry — if not found, return silently
2. Check entry `loggedAt` within backfill window — if not, return silently
3. Delete entry
4. Publish `ActivityEvent(type: deleted, definitionId, loggedAt, value: null)`

---

## Read Functions

### `getTotalLoggedForCommitmentOnDate(definitionId, date)`

Sum of all `value` fields for a commitment on a given date. Used by PerformanceService after each recalculation.

### `getEntriesForCommitment(definitionId, from, to)`

All log entries in a date range. Used by Analytics and LogHistorySheet.

### `getEntriesForDay(definitionId, date)`

All log entries for a specific day ordered by `loggedAt` descending. Used by LogHistorySheet.

---

## Event Subscriptions

### `InstancePermanentlyDeletedEvent`

Deletes all log entries for `event.definitionId`. ActivityService owns its own data and cleans it up when the commitment is permanently removed.

---

## Rules

- `recordEntry`, `editEntry`, `deleteEntry` are the only functions that write to ActivityRepository
- `loggedAt` always passed by caller in `recordEntry` — never defaulted to `now` internally
- `loggedAt` immutable after creation — `editEntry` never changes it
- Backfill window enforced on all three write functions
- No commitment state check — logging allowed regardless of commitment state
- No calls to any other service
- All downstream reactions through `ActivityEvent`

---

## Dependencies

- ActivityRepository — log entry storage
- EventBus — publishes ActivityEvent; subscribes to InstancePermanentlyDeletedEvent
- AppConfig — maxLogBackfillDays