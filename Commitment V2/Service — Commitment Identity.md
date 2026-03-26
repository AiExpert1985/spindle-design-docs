**File Name**: commitmentidentityservice **Feature**: Commitment **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** manages the lifecycle of commitment instances. The only service that creates or deletes instances. The sole subscriber to CommitmentEvent within the Commitment feature. Publishes the three instance events that form the public interface of the Commitment feature.

---

## Why This Service Is the Bridge

CommitmentService manages definitions and publishes internal events. CommitmentIdentityService translates those into instance operations and publishes instance events that the rest of the application subscribes to. No feature outside Commitment ever sees a CommitmentEvent — they only see instance events.

`livePerformance` on each instance is calculated and written by PerformanceService via `updateLivePerformance()` — a valid downward call. This service stores the value, never calculates it.

---

## How Instances Are Generated

All instances are daily regardless of recurrence type. This uniformity means PerformanceService treats every instance identically — no special-casing by recurrence type.

**Daily** — one instance per calendar day. Target = `target.value`.

**Weekly** — one instance per day, each carrying 1/7 of the weekly target. Example: weekly goal of 200 pages → daily instance with target 28.6 pages. The user can act on any day — over-performance on one day compensates for under-performance on another, and the daily overall score receives a fair contribution every day.

**SpecificWeekDays** — one instance per commitment, generated only on the configured days. Days not in the list produce no instance and expect no activity. Each instance carries the full `target.value` — the target applies per occurrence, not divided.

There is always exactly one pending instance per commitment. When the pending instance closes, its successor is generated immediately. Managing one pending instance at a time avoids the complexity of tracking multiple simultaneous pending instances for the same commitment.

---

## Self-Replication

When an instance window ends, close and replicate happen as sequential steps in the same tick handler — not through events. `_closeInstance()` sets status to closed, then `_generateNextInstance()` is called immediately. `InstanceUpdatedEvent` is published after both steps complete, carrying the new status. This avoids self-subscription and keeps the sequence atomic.

For SpecificWeekDays, `calculateNextWindowStart` uses the `weekDays` list from the `SpecificWeekDays` recurrence variant to find the next configured day. Example: for [Sun, Fri], a Sunday instance spawns a Friday successor; a Friday instance spawns a Sunday successor on the following week.

Replication uses only data from the closing instance. The definition is consulted only to confirm the commitment is still active.

---

## Recreation on Definition Change — One Pending Instance

Because there is always exactly one pending instance per commitment, recreation is simple: clear the single pending instance and generate a new one from the updated snapshot.

This is triggered by any `CommitmentEvent(type: updated)` — regardless of what changed (target, recurrence, activityWindow, warningEnabled, or state). The identity service does not inspect what specifically changed — it always clears and recreates. This uniformity is why a single `updated` event type with a full snapshot is sufficient: different changes would have produced the same reaction anyway.

Example: user changes SpecificWeekDays from [Sun, Tue] to [Sun, Mon]. The single pending instance (which might have been a Tuesday instance) is cleared. A new instance is generated for the correct next occurrence under the new configuration. No orphan detection needed.

Closed instances are never touched — they are permanent historical records.

---

## Events Subscribed

Each CommitmentEvent type has its own dedicated listener — one responsibility per listener, independently testable.

### `CommitmentEvent(type: created)` → `_onCreate(event)`

Generates the first instance from `event.snapshot`. Calculates the first window start based on recurrence type and today's date. Publishes `InstanceCreatedEvent`.

### `CommitmentEvent(type: updated)` → `_onUpdate(event)`

Calls `_clearPendingInstance(event.definitionId)` to remove the pending instance for this commitment only. If `snapshot.commitmentState == active`, calls `_generateInstance(event.definitionId, snapshot, ...)` to create a new one. If not active, nothing more.

All operations are scoped to `event.definitionId` — other commitments' instances are never touched.

### `CommitmentEvent(type: deleted)` → `_onDelete(event)`

Deletes all instances including closed history. Publishes `InstancePermanentlyDeletedEvent`.

### `LongIntervalTickEvent` → `_onTick(event)`

**Close and replicate:** for each pending instance whose `regenerationWindow.windowEnd <= now`, calls `_closeInstance()` then `_generateNextInstance()` sequentially. Publishes `InstanceUpdatedEvent` after both steps complete.

Closing is unconditional — it happens regardless of `commitmentState`. An instance whose window has ended is a historical fact; the commitment's current state has no bearing on whether that window closed. What the commitment state controls is replication only: if `commitmentState == active`, a successor instance is generated. If not active (frozen, completed, deleted), the instance closes and no successor is created.

**Catch-up:** for each commitment with `commitmentState == active` and no pending instance (device was inactive at close time), replicates from the last closed instance. Catch-up is restricted to active commitments — frozen, completed, and deleted commitments do not need a new pending instance. If no closed instance exists (brand new commitment that never had one), reads definition once and generates from scratch.

**Week boundary:** if `TemporalHelper.isWeekBoundary(tick.timestamp)` and TickGuard confirms not already run for this week boundary, publishes `WeekEndedEvent(previousWeekStart)` and `WeekStartedEvent(currentWeekStart)`. Week boundaries are user-relative — `TemporalHelper` uses `UserCoreService` to determine the correct week start day. This is published here rather than in Infrastructure because detecting the boundary requires user preferences, which infrastructure cannot access.

All tick operations are idempotent. TickGuard prevents double-processing.

---

## Events Published

```
InstanceCreatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
```

New pending instance exists. Triggers PerformanceService to initialise `livePerformance`.

```
InstanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  snapshot: CommitmentSnapshot
```

Structural change — commitmentState, status (including pending→closed), currentTarget, recurrence, activityWindow, or regenerationWindow. Does not fire for `livePerformance` changes. Carries a `CommitmentSnapshot` so subscribers have everything they need without a follow-up service call. `CommitmentSnapshot` is defined in `service_commitment` — see that doc.

```
InstancePermanentlyDeletedEvent
  definitionId: String
```

All instances for this commitment have been deleted. Subscribers clean up their own linked records.

---

## Core Functions

### `_generateInstance(definitionId, snapshot, windowStart, windowEnd)`

Creates one instance. `definitionId` is passed explicitly — it is the identity of the commitment, not carried in the snapshot. `RegenerationWindow` constructed from `windowStart` and `windowEnd`. `ActivityWindow` (including `warningEnabled`) copied from `snapshot.activityWindow`. Status: pending. `livePerformance`: `0.0` — valid ground state before any activity. PerformanceService receives `InstanceCreatedEvent` and recalculates immediately. Idempotent — exits silently if an instance already exists for `definitionId + windowStart`. Publishes `InstanceCreatedEvent`.

### `_clearPendingInstance(definitionId)`

Deletes the single pending instance for this commitment, if one exists. Closed instances are never touched. Used by `_onUpdate` and `_onDelete` before recreating or removing instances. Safe to call when no pending instance exists — exits silently.

### `_closeInstance(instanceId)`

Sets `status: closed`. Called only from `_onTick`, immediately before `_generateNextInstance()`.

### `_generateNextInstance(closedInstance)`

Calculates next `windowStart` via `calculateNextWindowStart()`. Generates successor instance. Called immediately after `_closeInstance()` in the same tick handler. `InstanceUpdatedEvent` is published after both complete.

### `updateLivePerformance(instanceId, value)`

Public write function called by PerformanceService. Writes `livePerformance`. Does not publish `InstanceUpdatedEvent` — livePerformance changes are signalled by `PerformanceUpdatedEvent`.

### `calculateNextWindowStart(instance)`

Pure function. Switches on `instance.recurrence`:

- `Daily` → `windowStart + 1 day`
- `Weekly` → `windowStart + 1 day`
- `SpecificWeekDays(weekDays)` → next day in `weekDays` after `windowStart`, wrapping to next week if needed

---

## Read Functions

### `getInstances(from, to, definitionId?)`

Universal reader. Returns all instances whose `regenerationWindow` falls within the date range, optionally filtered by commitment. All other read functions are wrappers.

### `watchInstancesForDay(date)`

Stream for live UI updates. Emits on any structural instance change for instances active on the given date.

### `getCurrentInstance(definitionId)`

The current pending instance for a commitment. Used by ActivityService before logging.

### `getInstanceForCommitmentOnDate(definitionId, date)`

Instance whose `regenerationWindow` contains the given date. Used by PerformanceService to identify which instance a log belongs to.

### `getInstancesForDay(date)`

All instances active on a given day across all commitments.

### `getInstancesForWeek(weekStart, definitionId?)`

All instances for a given week, optionally filtered by commitment.

---

## Rules

- The only service that creates or deletes instances
- `updateLivePerformance()` is the only public write function — called by PerformanceService only
- `livePerformance` changes do not trigger `InstanceUpdatedEvent`
- Close and replicate are sequential in the same tick handler — no self-subscription
- Closed instances never deleted except on permanent commitment deletion
- Exactly one pending instance per commitment at all times
- Commitment state checked before replication — non-active commitments do not get a successor
- Any `CommitmentEvent(type: updated)` clears and recreates the pending instance — no inspection of what changed
- CommitmentEvent is consumed only by this service — never forwarded outside Commitment

---

## Dependencies

- EventBus — subscribes to `CommitmentEvent` (internal), `LongIntervalTickEvent`; publishes `InstanceCreatedEvent`, `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`, `WeekEndedEvent`, `WeekStartedEvent`
- CommitmentInstanceRepository — instance storage
- CommitmentService — reads definition for catch-up generation when no prior instance exists
- TemporalHelper — day and week boundary detection
- TickGuard — prevents double-processing on tick