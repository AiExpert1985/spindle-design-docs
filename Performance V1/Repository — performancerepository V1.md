**Created**: 18-Mar-2026 **Modified**: - **Feature**: Performance **Phase**: 2 (Drift) · Phase 3 (Firestore)

**Purpose:** all data access for CommitmentDailyEntry records. Append-only by design — no update or delete operations except cascade delete on permanent commitment deletion. Abstract interface — nothing above this layer knows whether Drift or Firestore is active.

---

## Operations

| Operation                    | Input                  | Output                                              |
| ---------------------------- | ---------------------- | --------------------------------------------------- |
| `appendEntry`                | CommitmentDailyEntry   | —                                                   |
| `fetchEntriesForCommitment`  | definitionId, from, to | List of entries in range                            |
| `fetchEntriesForPeriod`      | from, to               | List of all entries in range across all commitments |
| `fetchLatestEntry`           | definitionId           | CommitmentDailyEntry? — most recent entry           |
| `deleteEntriesForCommitment` | definitionId           | — cascade delete only                               |

---

## Rules

- Never called directly by any feature other than `PerformanceAccountingService`
- `appendEntry` is the only write operation — no update operation exists by design
- `deleteEntriesForCommitment` called only by permanent commitment deletion — never individually
- All fetches use a date range — never fetch all entries for a commitment
- Firestore implementation scopes all queries to `/users/{userId}/performanceEntries/{id}` subcollection

---

## Firestore Path

```
/users/{userId}/performanceEntries/{id}
```

Add to database architecture subcollection list. Covered by existing security rule — no additional configuration needed.

---

## Query Pattern

The most frequent query is `fetchEntriesForCommitment` with a 7-day window for weekly cup calculation, and a rolling window for garment percent calculation. Both are simple date range queries on `windowStart` — one range filter, Firestore-compatible with no composite index needed beyond the single field index.