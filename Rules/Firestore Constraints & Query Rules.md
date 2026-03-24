
**Created**: 15-Mar-2026
**Modified**: -
**Phase:** applies from Phase 1 design · implemented in Phase 3
**Folder**: Rules

**Purpose:** documents the constraints that apply to FirestoreRepository. These constraints must be considered from Phase 1 — the Drift schema and repository interface must be designed to work within these limits so migration to Firestore has no surprises.

---

## Why This Matters in Phase 1

Drift has no query constraints — any filter combination works. Firestore does not. If you design queries in Phase 1 that require two range filters or server-side aggregations, those queries will break when you migrate to Firestore in Phase 3.

Design every repository query as if Firestore is already the backend.

---

## Query Constraints

**One range filter per query.** A Firestore query can have only one field with a range operator (`<`, `<=`, `>`, `>=`, `!=`). Your date range queries use one range on `windowStart` — compatible. Any additional filtering (by `status`, by `definitionId` equality) is done in Dart after fetching, never added as a second range to the query.

**No server-side aggregations.** No SUM, AVG, COUNT in Firestore queries. All aggregations — scores, averages, streak counts — are computed in Dart by the service layer after fetching documents.

**No joins.** Each collection is queried independently. Relations between collections (e.g. instances → definitions) are resolved in Dart, never in the query.

**Equality filters before range filters.** When combining equality and range filters, equality fields must come first in the index. Example: `definitionId == x AND windowStart >= from` — correct order.

---

## User Isolation — Subcollection Paths

All queries are scoped to the current user via the subcollection path. No `userId` field needed in documents or queries.

```dart
// Always scope to current user's subcollection
firestore
  .collection('users/$currentUserId/instances')
  .where('windowStart', '>=', from)
  .where('windowStart', '<=', to)
```

This means every query has at most one range filter — zero composite index costs on instance queries.

---

## Required Composite Indexes

Only three composite indexes needed for the entire app:

```
instances:   definitionId ASC, windowStart ASC
logEntries:  instanceId ASC, loggedAt ASC
logEntries:  definitionId ASC, loggedAt ASC
```

All other queries use single-field filters — Firestore handles those automatically with no composite index needed.

---

## Offline Persistence Rules

- Always enabled — never disable
- Streams read from local cache first, sync with server in background
- If a listener is disconnected for more than 30 minutes, Firestore charges for a full re-read on reconnect — this is acceptable for a daily-use app
- All analytical reads go against the local cache — never trigger a direct server read for analytics

---

## Batch Writes

Use batched writes for all multi-document operations. Firestore allows up to 500 operations per batch.

- Migration writes: batch in groups of 500, ramp up gradually (500/50/5 rule)
- Never write more than 500 operations per second to a new collection
- Instance generation (2 days ahead): batch write all new instances in one operation

---

## Security Rules Pattern

One rule covers all user data:

```
match /users/{userId}/{collection}/{docId} {
  allow read, write: if request.auth.uid == userId;
}
```

Queries must always match security rule constraints — Firestore rejects queries that could potentially return documents the user cannot read.

---

## Document Size

Keep documents small — under 100KB each. Your models are well within this limit. Log entries and instances are the smallest documents. Commitment definitions are slightly larger but still minimal.

Do not store arrays that grow unboundedly inside documents. Context tags are small fixed lists — fine. Never store all log entries inside an instance document.