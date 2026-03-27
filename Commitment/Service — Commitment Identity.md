**File Name**: service_commitment_identity **Feature**: Commitment **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** manages the lifecycle of commitment instances. The only service that creates or deletes instances. The sole subscriber to `CommitmentEvent` within the Commitment feature. Publishes the three instance events that form the public interface of the Commitment feature.

---

## Why This Service Is the Bridge

`CommitmentService` manages definitions and publishes internal events. `CommitmentIdentityService` translates those into instance operations and publishes instance events that the rest of the application subscribes to. No feature outside Commitment ever sees a `CommitmentEvent` — they only see instance events.

`livePerformance` on each instance is calculated and written by `PerformanceService` via `updateLivePerformance()` — a valid downward call. This service stores the value, never calculates it.

---

## How Instances Are Generated

All instances are daily regardless of recurrence type. This uniformity means `PerformanceService` treats every instance identically — no special-casing by recurrence type.

**Daily** — one instance per calendar day. Target = `target.value`.

**Weekly** — one instance per day, each carrying 1/7 of the weekly target. Example: weekly goal of 200 pages → daily instance with target 28.6 pages.

**SpecificWeekDays** — one instance per commitment, generated only on the configured days. Days not in the list produce no instance. Each instance carries the full `target.value`.

There is always exactly one pending instance per commitment. When the pending instance closes, its successor is generated immediately.

---

## Self-Replication

When an instance window ends, close and replicate happen as sequential steps in the same tick handler — not through events. `_closeInstance()` sets status to closed, then `_generateNextInstance()` is called immediately. `InstanceUpdatedEvent` is published after both steps complete. This avoids self-subscription and keeps the sequence atomic.

Replication uses only data from the closing instance. The definition is consulted only when no prior instance exists (catch-up on first launch after a brand new commitment) — not on every replication.

For `SpecificWeekDays`, `calculateNextWindowStart` uses the `weekDays` list to find the next configured day. Example: for [Sun, Fri], a Sunday instance spawns a Friday successor; a Friday instance spawns a Sunday successor on the following week.

---

## Recreation on Definition Change

Because there is always exactly one pending instance per commitment, recreation is simple: clear the single pending instance and generate a new one from the updated snapshot.

This is triggered by any `CommitmentEvent(type: updated)` — regardless of what changed. The identity service does not inspect what specifically changed — it always clears and recreates. This uniformity is why a single `updated` event type with a full snapshot is sufficient.

Example: user changes `SpecificWeekDays` from [Sun, Tue] to [Sun, Mon]. The single pending instance (which might have been a Tuesday instance) is cleared. A new instance is generated for the correct next occurrence under the new configuration. No orphan detection needed.

Closed instances are never touched — they are permanent historical records.

---

## Events Subscribed

### Internal stream — `CommitmentEvent(type: created)` → `_onCreate(event)`

Generates the first instance from `event.snapshot`. Calculates the first window start based on recurrence type and today's date. Publishes `InstanceCreatedEvent`.

### Internal stream — `CommitmentEvent(type: updated)` → `_onUpdate(event)`

Calls `_clearPendingInstance(event.definitionId)`. If `snapshot.commitmentState == active`, calls `_generateInstance(...)` to create a new one. If not active, nothing more. All operations scoped to `event.definitionId` only.

### Internal stream — `CommitmentEvent(type: deleted)` → `_onDelete(event)`

Deletes all instances including closed history. Publishes `InstancePermanentlyDeletedEvent`.

### `Heartbeat.longIntervalTick` → `_onTick(tick)`

**Close and replicate:** for each pending instance whose `regenerationWindow.windowEnd <= now`, calls `_closeInstance()` then `_generateNextInstance()` sequentially. Publishes `InstanceUpdatedEvent` after both steps complete.

Closing is unconditional — it happens regardless of `commitmentState`. What `commitmentState` controls is replication only: if `commitmentState == active`, a successor is generated. If not active, the instance closes and no successor is created.

**Catch-up:** for each commitment with `commitmentState == active` and no pending instance (device was inactive at close time), replicates from the last closed instance. If no closed instance exists, reads definition once and generates from scratch.

All tick operations use `TickGuard` to prevent double-processing.

---

## Events Published

```
InstanceCreatedEvent
  instanceId: String
  definitionId: String
  name: String
  windowStart: DateTime
```

New pending instance exists. Carries `name` so subscribers never need a definition lookup. Triggers `PerformanceService` to initialise `livePerformance`.

```
InstanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  snapshot: CommitmentSnapshot
```

Structural change — commitmentState, status (including pending→closed), currentTarget, recurrence, activityWindow, or regenerationWindow. Does not fire for `livePerformance` changes. Carries a `CommitmentSnapshot` so subscribers have everything they need without a follow-up call. `CommitmentSnapshot` is defined in `service_commitment`.

```
InstancePermanentlyDeletedEvent
  definitionId: String
```

All instances for this commitment have been deleted. Subscribers clean up their own linked records.

---

## Core Functions

### `_generateInstance(definitionId, snapshot, windowStart, windowEnd)`

Creates one instance. `RegenerationWindow` constructed from `windowStart` and `windowEnd`. `ActivityWindow` (including `warningEnabled`) copied from `snapshot.activityWindow`. Status: pending. `livePerformance`: `0.0`. Idempotent — exits silently if an instance already exists for `definitionId + windowStart`. Publishes `InstanceCreatedEvent` with `name` from snapshot.

### `_clearPendingInstance(definitionId)`

Deletes the single pending instance for this commitment. Closed instances never touched. Safe to call when no pending instance exists.

### `_closeInstance(instanceId)`

Sets `status: closed`. Called only from `_onTick`, immediately before `_generateNextInstance()`.

### `_generateNextInstance(closedInstance)`

Calculates next `windowStart` via `calculateNextWindowStart()`. Generates successor instance. Called immediately after `_closeInstance()`. `InstanceUpdatedEvent` published after both complete.

### `updateLivePerformance(instanceId, value)`

Public write function called by `PerformanceService` only. Writes `livePerformance`. Does not publish `InstanceUpdatedEvent`.

### `calculateNextWindowStart(instance)`

Pure function. Switches on `instance.recurrence`:

- `Daily` → `windowStart + 1 day`
- `Weekly` → `windowStart + 1 day`
- `SpecificWeekDays(weekDays)` → next day in `weekDays` after `windowStart`, wrapping to next week if needed

---

## Read Functions

### `getInstances(from, to, definitionId?)`

Universal reader. Returns all instances whose `regenerationWindow` falls within the date range, optionally filtered by commitment.

### `watchInstancesForDay(date)`

Stream for live UI updates. Emits on any structural instance change for instances active on the given date.

### `getCurrentInstance(definitionId)`

The current pending instance for a commitment. Used by `ActivityService` before logging.

### `getInstanceForCommitmentOnDate(definitionId, date)`

Instance whose `regenerationWindow` contains the given date. Used by `PerformanceService`.

### `getInstancesForDay(date)`

All instances active on a given day across all commitments.

### `getInstancesForWeek(weekStart, definitionId?)`

All instances for a given week, optionally filtered by commitment.

---

## Dependencies

- Internal stream — subscribes to `CommitmentEvent` published by `CommitmentService`
- `Heartbeat` — subscribes to `longIntervalTick`
- `CommitmentInstanceRepository` — instance storage
- `CommitmentService` — reads definition for catch-up generation when no prior instance exists
- `TemporalHelperService` — day boundary detection for catch-up logic
- `TickGuard` (Domain utility) — prevents double-processing on tick

---

## Rules

- The only service that creates or deletes instances
- `updateLivePerformance()` is the only public write function — called by `PerformanceService` only
- `livePerformance` changes do not trigger `InstanceUpdatedEvent`
- Close and replicate are sequential in the same tick handler — no self-subscription
- Closed instances never deleted except on permanent commitment deletion
- Exactly one pending instance per commitment at all times
- Commitment state checked before replication — non-active commitments do not get a successor
- Any `CommitmentEvent(type: updated)` clears and recreates the pending instance — no inspection of what changed
- `CommitmentEvent` consumed only by this service — never forwarded outside Commitment
- Week and day boundary events are published by `TemporalHelperService` — never by this service