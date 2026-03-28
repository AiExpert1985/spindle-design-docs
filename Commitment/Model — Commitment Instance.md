**File Name**: model_commitment_instance **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** one occurrence of a commitment on a specific date. Carries a DNA snapshot of the definition fields at creation time — everything any feature needs to act on it without a definition lookup.

---

## Status

```
pending → closed
```

**pending** — the window is open. The user can log activity.

**closed** — the window has ended. The instance is a permanent record.

One-way by design. A closed instance is a historical fact — making it reversible would undermine the integrity of records other features rely on.

When status transitions from `pending` to `closed`, `CommitmentIdentityService` publishes `InstanceUpdatedEvent`. Features that react to window close subscribe to this event and check `instance.status == closed`.

Closed instances are immutable except for `livePerformance`.

---

## Instance Deletion

Closed instances are permanent — removed only when the commitment is permanently deleted.

Pending instances are transient — there is always exactly one, and it is replaced when the commitment changes or when its window closes and a successor is generated. On soft delete, the pending instance is cleared — closed instances are preserved. The pending instance is recreated when the commitment is restored to active.

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

ActivityWindow
  startMinutes: int
  durationMinutes: int
  warningEnabled: bool
```

- **id** — unique identifier. Immutable.
- **definitionId** — the commitment this instance belongs to.
- **name** — the commitment's name. Carried here so notifications and upper features can display it without a definition lookup.
- **commitmentType** — do or avoid. Carried from the definition so performance calculation never needs the definition.
- **recurrence** — used to calculate the next window start when this instance spawns its successor.
- **regenerationWindow** — the accountability period. `windowStart` and `windowEnd` are immutable after creation.
- **activityWindow** — the user's preferred time-of-day slot and notification settings. Governs notification timing only — the instance always lives its full `regenerationWindow` duration regardless.
- **currentTarget** — the target in effect for this window. Reflects the definition's target at creation time. On a pending instance, updated when the definition target changes so the pending window always reflects the current intent. Closed instances are never modified — their target is the historical fact of what was in effect at the time.
- **commitmentState** — the commitment's current lifecycle state. On a pending instance, updated by `CommitmentIdentityService` whenever the commitment state changes. Closed instances are never modified — their state is preserved as it was when the window closed.
- **status** — pending or closed. One-way transition. Change from pending to closed triggers `InstanceUpdatedEvent`.
- **livePerformance** — how much of the target was achieved, as a percentage. Maintained by `PerformanceService`. Do: can exceed 100%. Avoid: capped at 100%. Does not trigger `InstanceUpdatedEvent` when it changes.
- **createdAt** — immutable.
- **updatedAt** — updated on every structural write. Not updated when only `livePerformance` changes.

---

## Rules

- `CommitmentIdentityService` is the only service that creates or deletes instances
- `PerformanceService` is the only service that writes `livePerformance`
- `regenerationWindow`, `recurrence`, `commitmentType`, `createdAt` are immutable after creation
- Closed instances are immutable — the only exception is `livePerformance`, which can be updated retroactively to allow the user to log forgotten activity
- Definition changes (target, state, recurrence) apply to the pending instance only — closed instances are never modified
- `livePerformance` changes do not trigger `InstanceUpdatedEvent` — signalled by `PerformanceUpdatedEvent`
- `livePerformance` is initialized to `0.0` on creation — valid ground state before any activity
- Status change from pending to closed always triggers `InstanceUpdatedEvent`
- On soft delete, the pending instance is cleared — closed instances are preserved