**File Name**: gracerepository **Feature**: Grace **Phase**: 2 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage interface for the Grace feature. Stores `GraceRecord` entries. Called only by `GraceService`.

---

## Operations

### `saveGraceRecord(record)`

Upsert. Creates or updates a grace record.

### `getGraceRecord(definitionId, windowDate)`

Returns the grace record for a specific commitment window, or null if no grace was granted. Identified by `definitionId + windowDate` — consistent with the model fields.

### `markFired(id)`

Sets `fired: true` on the record. Called immediately after the follow-up notification is scheduled.

### `deleteRecordsForCommitment(definitionId)`

Deletes all grace records for one commitment. Called on permanent commitment deletion.

---

## Rules

- Called only by `GraceService`
- One record per commitment window — enforced at the service layer, not the repository
- No business logic — only fetch and store