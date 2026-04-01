**File Name**: service_commitment **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 30-Mar-2026

---

**Purpose:** manages the lifecycle of commitment definitions. The single writer of `CommitmentDefinition`. Handles creation, editing, and all state transitions. Publishes `CommitmentEvent` consumed only by `CommitmentIdentityService` within the same feature.

---

## CommitmentEvent — Internal Only

```
CommitmentEvent
  type: CommitmentEventType
  definitionId: String
  snapshot: CommitmentSnapshot?   // null on deleted
```

```
enum CommitmentEventType { created, updated, deleted }
```

```
CommitmentSnapshot
  name: String
  commitmentType: CommitmentType
  recurrence: Recurrence
  target: Target
  activityWindow: ActivityWindow   // includes warningEnabled
  commitmentState: CommitmentState
```

**Why one event type with three enum values rather than separate events per change?**

`CommitmentIdentityService` always performs the same operation for any update: clear the pending instance and recreate it from the snapshot. Whether the target changed, the recurrence changed, or the state transitioned — the reaction is identical. One `updated` event with a full snapshot is simpler and equally expressive. Adding a new editable field requires no new event type.

The three enum values map to three genuinely different operations:

- `created` — generate the first instance
- `updated` — clear and recreate the pending instance
- `deleted` — delete all instances permanently

`updated` covers all definition changes including state transitions. Soft delete is a state change — published as `updated` with `snapshot.commitmentState == deleted`. Only permanent deletion uses `deleted`.

`snapshot` carries everything `CommitmentIdentityService` needs to recreate instances — no follow-up definition lookup required.

---

## Write Functions

### `createCommitment(definition)`

Validates and saves with `commitmentState: active`. Writes first `StateTransitionRecord`. Publishes `CommitmentEvent(type: created, snapshot: ...)`.

Validation: `name` non-empty, `target.value >= 0`, `SpecificWeekDays` only if `weekDays` non-empty.

### `updateCommitment(definition)`

Saves edited definition. Rejects `commitmentType` changes — immutable after creation. Publishes `CommitmentEvent(type: updated, snapshot: ...)`.

### `freezeCommitment(id)`

Valid from `active` only. Sets state to `frozen`. Writes `StateTransitionRecord`. Publishes `CommitmentEvent(type: updated, snapshot: ..., commitmentState: frozen)`.

### `unfreezeCommitment(id)`

Valid from `frozen` only. Sets state to `active`. Writes `StateTransitionRecord`. Publishes `CommitmentEvent(type: updated, snapshot: ..., commitmentState: active)`.

### `completeCommitment(id)`

Valid from `active` only. Sets state to `completed`. Writes `StateTransitionRecord`. Publishes `CommitmentEvent(type: updated, snapshot: ..., commitmentState: completed)`.

### `deleteCommitment(id)`

Soft delete. Sets state to `deleted`. Writes `StateTransitionRecord`. Publishes `CommitmentEvent(type: updated, snapshot: ..., commitmentState: deleted)`.

### `restoreCommitment(id)`

Valid from `deleted` only. Sets state to `active`. Writes `StateTransitionRecord`. Publishes `CommitmentEvent(type: updated, snapshot: ..., commitmentState: active)`.

### `permanentDeleteCommitment(id)`

Valid from `deleted` only. Removes definition and full `StateTransitionLog`. Publishes `CommitmentEvent(type: deleted, snapshot: null)`.

---

## State Transition Table

|From|To|Function|
|---|---|---|
|—|active|createCommitment|
|active|frozen|freezeCommitment|
|active|completed|completeCommitment|
|active|deleted|deleteCommitment|
|frozen|active|unfreezeCommitment|
|frozen|deleted|deleteCommitment|
|completed|deleted|deleteCommitment|
|deleted|active|restoreCommitment|
|deleted|permanent|permanentDeleteCommitment|

Delete is available from any non-deleted state. Invalid transitions return a `Failure` result — never silently ignored.

---

## Read Functions

**Streams**

- `watchActiveCommitments()` — dashboard, commitments screen
- `watchFrozenCommitments()` — frozen filter
- `watchDeletedCommitments()` — recycle bin
- `watchCompletedCommitments()` — Your Record

**One-time**

- `getDefinition(id)` — used when display data is needed (description, full details)
- `getCommitmentCountsByState()` — counts grouped by state: `{ active, frozen, completed, deleted }`. Used by dashboard filter chips and Your Record.
- `getPortfolioSize()` — total count excluding permanently deleted. Used by `UserCapabilityService` to enforce free tier limits.
- `getActiveCount()` — count of active commitments only.
- `getRecentlyCreated(since)` — definitions created after a given date. Used by rate limit checks.
- `getStateTransitionLog(definitionId)` — full ordered log for one commitment.
- `getActivityWindowDefaults(recurrence) → ActivityWindowDefaults` — returns the default `startMinutes` and `durationMinutes` for a given recurrence type. Reads the user's temporal preferences from `UserCoreService` and maps them to the correct defaults per recurrence. Called by the commitment form to pre-fill the activity window.

---

## Rules

- The only writer of `CommitmentDefinition`
- `CommitmentEvent` is consumed only by `CommitmentIdentityService` — never by features outside Commitment
- Never calls `CommitmentIdentityService` — instances react through the internal event stream
- `permanentDeleteCommitment` publishes `CommitmentEvent(type: deleted)` only — calls nothing else
- `commitmentType` changes rejected on update — immutable after creation
- All functions return `Result<T>` — no raw exceptions

---

## Dependencies

- `CommitmentRepository` — definition and state transition log storage
- `UserCoreService` — reads temporal preferences for `getActivityWindowDefaults()`
- Internal stream — publishes `CommitmentEvent` consumed only by `CommitmentIdentityService`