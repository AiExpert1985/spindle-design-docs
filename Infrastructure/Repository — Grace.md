**File Name**: gracerepository
**Feature**: Infrastructure
**Phase**: 1
**Created**: 15-Mar-2026
**Modified**: 20-Mar-2026

---

**Purpose:** storage interface for the Grace feature. Stores `GraceRecord` entries. Called only by `GraceService`.

---

## Operations

### `saveGraceRecord(record)`

Upsert. Creates or updates a grace record.

### `getGraceRecord(instanceId)`

Returns the grace record for a specific instance. Returns null if no grace was granted.

### `getExpiredUnfiredRecords(now)`

Returns all grace records where `graceUntil <= now` and `fired == false`. Used by `GraceService` on tick to find expired grace periods that need follow-up notifications.

### `deleteRecordsForCommitment(definitionId)`

Deletes all grace records for one commitment. Called on permanent commitment deletion.

---

## Rules

- Called only by `GraceService`
- One record per instance — enforced at the service layer, not the repository
- No business logic — only fetch and store