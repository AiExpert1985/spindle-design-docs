**File Name**: activityrepository **Feature**: Activity **Phase**: 1 (Drift) · Phase 3 (Firestore) **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** storage interface for LogEntry records. Called only by ActivityService. Abstract interface — nothing above this layer knows whether Drift or Firestore is active.

---

## Operations

### `saveEntry(entry)`

Creates a new log entry. Append-only — never called for updates.

### `updateEntry(entryId, value, note?, updatedAt)`

Updates `value`, `note`, and `updatedAt` on an existing entry. Only these three fields are ever updated — `loggedAt`, `definitionId`, and `createdAt` are immutable.

### `deleteEntry(entryId)`

Deletes one log entry by id. Only called within the backfill window — ActivityService enforces the restriction before calling.

### `getEntriesForDay(definitionId, date)`

All log entries for a commitment on a specific day, ordered by `loggedAt` ascending.

### `getEntriesForPeriod(definitionId, from, to)`

All log entries for a commitment in a date range. Used by Analytics and LogHistorySheet.

### `getTotalLoggedForCommitmentOnDate(definitionId, date)`

Sum of all `value` fields for a commitment on a given date. Used by PerformanceService.

### `deleteEntriesForCommitment(definitionId)`

Deletes all log entries for a commitment. Called by ActivityService on permanent commitment deletion only.

---

## Rules

- Called only by ActivityService
- `saveEntry` is append-only — never used for updates
- `updateEntry` changes only `value`, `note`, `updatedAt` — all other fields immutable
- `deleteEntry` called only after ActivityService confirms entry is within backfill window
- Bulk delete only on permanent commitment deletion
- No instanceId on LogEntry — all queries use `definitionId` + date
- No business logic — only fetch and store