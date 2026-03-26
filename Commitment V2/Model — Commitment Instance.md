**File Name**: commitmentinstance **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** one occurrence of a commitment on a specific date. Tracks what was achieved during that window. Carries everything any feature needs to act on it — no feature outside Commitment reads the definition for operational decisions.

---

## What an Instance Is

A commitment definition says what the user wants to do and how often. An instance is one actual occurrence — "did the user do it on this specific day?"

Instances are self-replicating. Each instance carries enough data to spawn its successor when it closes, without consulting the definition. This eliminates a central scheduler and makes the system resilient — a missed tick does not permanently break the chain.

All instances are daily regardless of recurrence type:

- **Daily** — one instance per calendar day, carrying the full daily target.
- **Weekly** — one instance per day, each carrying 1/7 of the weekly target. A weekly goal of 200 pages produces a daily instance with target 28.6 pages. The user can act on any day — over-performance on one day compensates for under-performance on another.
- **SpecificWeekDays** — one instance per commitment, generated only on the configured days. On non-configured days, no instance exists and no activity is expected. Each instance carries the full target per occurrence.

This uniformity makes performance calculation simple — every instance is treated identically regardless of how the commitment was configured.

There is always exactly one pending instance per commitment. When a pending instance closes, its successor is generated immediately in the same operation. This keeps the invariant clean and avoids complexity from managing multiple simultaneous pending instances.

The instance is the single operational record for any commitment occurrence. It carries the commitment's current state, name, and notification config so features above it never need to look up the definition.

`livePerformance` is the one field owned by an external feature (Performance). It changes independently of structural instance changes and does not trigger `InstanceUpdatedEvent` — it has its own signal (`PerformanceUpdatedEvent`).

---

## Two Windows

An instance has two distinct windows serving different purposes.

**RegenerationWindow** — the accountability period. Defines when this occurrence begins and ends. Used to calculate the next window start when the instance self-replicates. Immutable after creation.

**ActivityWindow** — the user's preferred time-of-day slot. Used for notification timing: the warning fires at 3/4 of the duration (if enabled), the window-close notification fires at the end. No effect on scoring or the accountability period.

Keeping these as separate named classes makes their distinct purposes immediately clear from the field names alone.

---

## Status

```
pending → closed
```

**pending** — the window is open. The user can log activity.

**closed** — the window has ended. The instance is a permanent record.

One-way by design. A closed instance is a historical fact — making it reversible would undermine the integrity of records other features (scores, streaks, calendar) rely on.

When status transitions from `pending` to `closed`, CommitmentIdentityService publishes `InstanceUpdatedEvent`. Features that react to window close — PerformanceService, NotificationSchedulerService — subscribe to this event and check `instance.status == closed` to identify the transition.

Closed instances are immutable except for `livePerformance` — which may update if a grace-period log arrives after window close.

---

## Instance Deletion

Closed instances are permanent — removed only when the commitment is permanently deleted.

Pending instances are transient — there is always exactly one, and it is replaced when the commitment changes or when its window closes and a successor is generated.

Closed = history. Pending = the current active occurrence.

---

## Fields

```
CommitmentInstance
  id: String
  definitionId: String
  name: String
  commitmentType: CommitmentType
  recurrence: Recurrence
  regenerationWindow: RegenerationWindow
  activityWindow: ActivityWindow
  currentTarget: Target
  commitmentState: CommitmentState
  status: InstanceStatus
  livePerformance: double
  createdAt: DateTime
  updatedAt: DateTime
```

**Supporting classes:**

```
RegenerationWindow
  windowStart: DateTime
  windowEnd: DateTime
```

```
ActivityWindow
  startMinutes: int
  durationMinutes: int
  warningEnabled: bool
```

---

- **id** — unique identifier. Immutable.
- **definitionId** — the commitment this instance belongs to.
- **name** — the commitment's name. Carried here so notifications and logs can display it without a definition lookup. Example: "Morning Run".
- **commitmentType** — do or avoid. Carried from the definition so performance calculation never needs the definition.
- **recurrence** — used to calculate the next window start when this instance spawns its successor. For SpecificWeekDays, the sealed class carries the configured days directly.
- **regenerationWindow** — the accountability period. `windowStart` and `windowEnd` are immutable after creation.
- **activityWindow** — the user's preferred time-of-day slot and notification settings. `warningEnabled` lives here because it governs notification behaviour for this window specifically. The default spans the full day (from `dayBoundaryHour` to the next `dayBoundaryHour`), meaning no time restriction. The user narrows it only when they want to force a specific slot — for example, morning exercise from 08:00–10:00. The activity window governs notification timing only; the instance always lives its full `regenerationWindow` duration regardless.
- **currentTarget** — the target in effect for this window. Updated when the definition target changes so performance is always calculated against the correct value.
- **commitmentState** — the commitment's current lifecycle state. Updated by CommitmentIdentityService whenever the commitment changes. Features read this instead of querying the definition. Example: ActivityService checks `instance.commitmentState == active` before accepting a log.
- **status** — pending or closed. One-way transition. Change from pending to closed triggers `InstanceUpdatedEvent`.
- **livePerformance** — how much of the target was achieved, as a percentage. Maintained by PerformanceService. Do: can exceed 100%. Avoid: capped at 100%. Does not trigger `InstanceUpdatedEvent` when it changes.
- **createdAt** — immutable.
- **updatedAt** — updated on every structural write. Not updated when only `livePerformance` changes.

---

## Rules

- CommitmentIdentityService is the only service that creates or deletes instances
- PerformanceService is the only service that writes `livePerformance`
- `regenerationWindow`, `recurrence`, `commitmentType`, `createdAt` are immutable after creation
- `livePerformance` changes do not trigger `InstanceUpdatedEvent` — signalled by `PerformanceUpdatedEvent`
- Exactly one pending instance per commitment at all times
- `livePerformance` is initialized to `0.0` on creation. PerformanceService receives `InstanceCreatedEvent` and immediately recalculates from logged activity — the `0.0` is a valid ground state before any activity exists, not a permanent value
- Status change from pending to closed always triggers `InstanceUpdatedEvent`