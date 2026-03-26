**File Name**: commitmentservice **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** manages the lifecycle of commitment definitions. The single writer of CommitmentDefinition. Handles creation, editing, and all state transitions. Publishes events consumed only within the Commitment feature.

---

## Public Interface of the Commitment Feature

The public interface of Commitment is the instance events published by CommitmentIdentityService — not CommitmentEvent. No feature outside Commitment subscribes to CommitmentEvent. It is an internal coordination mechanism between CommitmentService and CommitmentIdentityService within the same feature.

The rest of the application interacts with Commitment entirely through instances. Features read instances to get commitment name, state, target, and notification config — no definition lookup needed for operational decisions.

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

CommitmentIdentityService always performs the same operation for any update: clear the pending instance and recreate it from the snapshot. Whether the target changed, the recurrence changed, the activity window changed, or the state transitioned — the reaction is identical. Separate events (TargetChangedEvent, RecurrenceChangedEvent, StateChangedEvent) would all trigger the same handler with the same result. One `updated` event with a full snapshot is simpler, equally expressive, and easier to extend — adding a new editable field requires no new event type.

The three enum values map to three genuinely different operations:

- `created` — generate the first instance
- `updated` — clear and recreate the pending instance
- `deleted` — delete all instances permanently

`updated` covers all definition changes including state transitions. Soft delete is a state change — published as `updated` with `snapshot.commitmentState == deleted`. Only permanent deletion uses `deleted`.

`snapshot` carries everything CommitmentIdentityService needs to recreate instances — no follow-up definition lookup required. `ActivityWindow` in the snapshot includes `warningEnabled`.

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

Delete is available from any non-deleted state — active, frozen, or completed. A user must always be able to remove a commitment regardless of its current lifecycle stage. Invalid transitions return an error — never silently ignored.

---

## Read Functions

**Streams**

- `watchActiveCommitments()` — dashboard, commitments screen
- `watchFrozenCommitments()` — frozen filter
- `watchDeletedCommitments()` — recycle bin
- `watchCompletedCommitments()` — Your Record

**One-time**

- `getDefinition(id)` — used when display data is needed (description, full details)
- `getCommitmentsByState()` — counts grouped by state: `{ active, frozen, completed, deleted }`. Used by dashboard filter chips and Your Record.
- `getPortfolioSize()` — total count of all commitments excluding permanently deleted. Used by CommitmentAccessService to enforce free tier limits.
- `getActiveCount()` — count of active commitments only. Used by CommitmentAccessService for the active portfolio limit check.
- `getRecentlyCreated(since)` — definitions created after a given date. Used by rate limit checks.
- `getStateTransitionLog(definitionId)` — full ordered log for one commitment. Used by calendar and Your Record.

---

## Rules

- The only writer of CommitmentDefinition
- `CommitmentEvent` is consumed only by CommitmentIdentityService — never by features outside Commitment
- Never calls CommitmentIdentityService — instances react through events
- `permanentDeleteCommitment` publishes `CommitmentEvent(type: deleted)` only — calls nothing else
- `commitmentType` changes rejected on update — immutable after creation

**Why CommitmentService never closes instances directly:**

When a commitment is frozen, completed, or soft-deleted, the pending instance is not closed by CommitmentService. This is intentional. An instance's window is a fact about time — it closes when its `regenerationWindow.windowEnd` arrives, not when the commitment changes state. CommitmentService owns definitions. CommitmentIdentityService owns instances. CommitmentService may not call CommitmentIdentityService directly — CommitmentIdentityService is built above CommitmentService in the intra-feature dependency order, and lower services never call upward.

The correct flow: CommitmentService publishes `CommitmentEvent(type: updated, snapshot.commitmentState: frozen)`. CommitmentIdentityService subscribes, clears the pending instance (meaning no successor will be generated), and lets the existing pending instance close naturally when its window ends. The instance remains open and loggable until its `regenerationWindow.windowEnd` — the commitment state change does not retroactively invalidate the current window.

---

## Dependencies

- CommitmentRepository — definition and state transition log storage
- EventBus — publishes CommitmentEvent (internal)