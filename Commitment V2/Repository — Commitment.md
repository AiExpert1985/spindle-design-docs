**File Name**: commitmentrepository **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** storage interface for the Commitment feature. Stores CommitmentDefinition records and StateTransitionRecord entries. Called only by CommitmentService.

---

## CommitmentDefinition Operations

- `saveDefinition(definition)` — upsert. Sets `updatedAt` on every call.
- `getDefinition(id)` — returns one definition, null if not found.
- `watchDefinitionsByState(state)` — stream of all definitions matching `commitmentState`. Emits on any change.
- `getDefinitionsByState(state)` — one-time fetch. Used for bulk operations.
- `getDefinitionCount()` — total count
- `getDefinitionsCreatedSince(date)` — definitions with `createdAt >= date`. Used by rate limit checks.
- `deleteDefinition(id)` — hard delete. Only reachable through `permanentDeleteCommitment`.

---

## StateTransitionRecord Operations

- `appendTransition(record)` — append-only write. Never updated or deleted individually.
- `getTransitionLog(definitionId)` — full log ordered by timestamp ascending.
- `deleteTransitionLog(definitionId)` — deletes all records for one commitment. Only on permanent deletion.

---

## Rules

- Called only by CommitmentService
- No business logic — only fetch and store
- All IDs are client-generated UUIDs
- Hard delete only reachable through CommitmentService permanent deletion path