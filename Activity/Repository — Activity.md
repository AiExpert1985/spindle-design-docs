**File Name**: repository_activity **Feature**: Activity **Phase**: 1 (Drift) · Phase 3 (Firestore) **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** storage interface for `LogEntry` records. Called only by `ActivityService`. Abstract interface — nothing above this layer knows whether Drift or Firestore is active.

---

## Interface

```dart
abstract class ActivityRepository {
  Future<void> saveEntry(LogEntry entry);
  Future<void> updateEntry(String entryId, double value, String? note, DateTime updatedAt);
  Future<void> deleteEntry(String entryId);
  Future<LogEntry?> getEntry(String entryId);
  Future<List<LogEntry>> getEntriesForDay(String definitionId, DateTime date);
  Future<List<LogEntry>> getEntriesForWeek(String definitionId, DateTime weekStart);
  Future<List<LogEntry>> getEntriesForPeriod(String definitionId, DateTime from, DateTime to);
  Future<void> deleteEntriesForCommitment(String definitionId);
}
```

`saveEntry` — append-only. Never called for updates.

`updateEntry` — updates `value`, `note`, and `updatedAt` only. All other fields are immutable and never written here.

`deleteEntry` — hard delete for one entry. `ActivityService` enforces the backfill window before calling.

`getEntry` — returns one entry by ID, or null if not found. Used by `ActivityService` before edit or delete to confirm the entry exists and check its `loggedAt`.

`getEntriesForDay` — all entries for a commitment on a specific date, ordered by `loggedAt` ascending.

`getEntriesForWeek` — all entries for a commitment in the Mon–Sun week containing `weekStart`, ordered by `loggedAt` ascending.

`getEntriesForPeriod` — all entries for a commitment between `from` and `to` inclusive, ordered by `loggedAt` ascending. Used by Analytics.

`deleteEntriesForCommitment` — deletes all entries for a commitment. Called only on permanent commitment deletion.

---

## Firestore Compatibility

All queries use a single range filter on `loggedAt` combined with an equality filter on `definitionId` — compatible with Firestore's one-range-filter constraint. The composite index `definitionId ASC, loggedAt ASC` covers all read operations.

No aggregations in the repository — all totals and sums are computed by `ActivityService` after fetching entries.

---

## Rules

- Called only by `ActivityService`
- `saveEntry` is append-only — never used for updates
- `updateEntry` changes only `value`, `note`, `updatedAt` — all other fields immutable
- `deleteEntry` called only after `ActivityService` confirms the entry is within the backfill window
- Bulk delete only on permanent commitment deletion
- No business logic — only fetch and store
- No aggregations — raw records only