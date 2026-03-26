**File Name**: notificationschedulerservice **Feature**: Notifications **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** owns all notification timing decisions for commitment instances. Schedules and cancels notifications in response to instance lifecycle events. The single location within the Commitment feature for "when should a notification fire" logic. Calls `NotificationService` downward to schedule delivery — never sends to the OS directly.

**Why this belongs in the Commitment feature, not Infrastructure:**

This service subscribes to `InstanceCreatedEvent`, `InstanceUpdatedEvent`, and `InstancePermanentlyDeletedEvent` — all Commitment feature events. Infrastructure is feature-agnostic and may never subscribe to feature events (see `architecture_rules` section 11). Notification scheduling is commitment-specific business logic: it knows what an instance is, what `commitmentState` means, and what `activityWindow` represents. That knowledge belongs in the Commitment feature. `NotificationService` provides the delivery mechanism — this service decides when and what to deliver.

---

## The Notification Gate

Every scheduling decision passes through `_shouldNotify(instance)` before any notification is scheduled.

```
_shouldNotify(instance) → bool
  returns true only if instance.commitmentState == active
```

Frozen, completed, and soft-deleted commitments do not receive notifications. Applied on every `InstanceUpdatedEvent` and before every scheduling decision. If a commitment transitions to non-active mid-schedule, the next update cancels pending notifications.

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onInstanceCreated(event)`

New instance created. Reads instance via `CommitmentIdentityService.getCurrentInstance(definitionId)`. Calls `_shouldNotify(instance)` — if false, exits. If true, schedules warning and window-close notifications.

### `InstanceUpdatedEvent` → `_onInstanceUpdated(event)`

Instance changed. Reads updated instance from `event.snapshot`.

- If `_shouldNotify` false — calls `NotificationService.cancel()` for all pending notification IDs for this instance.
- If `_shouldNotify` true — reschedules notifications from the current instance state. Handles activity window changes and any other structural updates in one pass.

### `InstancePermanentlyDeletedEvent` → `_onInstanceDeleted(event)`

Calls `NotificationService.cancel()` for all notification IDs associated with this `definitionId`. Permanent — no reschedule.

---

## Notification Types

### Warning notification

Fires at 3/4 of the `activityWindow` duration, subject to minimum lead time.

- Time remaining > `AppConfig.minimumWarningLeadMinutes` → schedule via `NotificationService.schedule()`
- Time remaining ≤ `AppConfig.minimumWarningLeadMinutes` → skip — not enough time to be useful

Only scheduled if `instance.activityWindow.warningEnabled == true`.

Payload:

```
ThinNotification(
  id: 'warning_${definitionId}_${windowStart}',
  title: instance.name,
  body: 'Your window closes in {formatted duration}.',
)
```

### Window-close notification

Fires at `activityWindow.startMinutes + activityWindow.durationMinutes`.

Payload varies by `commitmentType`:

**Binary (target == 1):**

```
RichNotification(
  id: 'close_${definitionId}_${windowStart}',
  title: instance.name,
  body: 'Your window just closed.',
  actions: [
    NotificationAction(id: 'log_done',    label: 'Done',            params: {definitionId, windowDate}),
    NotificationAction(id: 'grant_grace', label: '15 more minutes', params: {definitionId, windowDate}),
    NotificationAction(id: 'dismiss',     label: 'Dismiss',         params: {}),
  ]
)
```

**Numerical:**

```
RichNotification(
  id: 'close_${definitionId}_${windowStart}',
  title: instance.name,
  body: 'Your window just closed.',
  actions: [
    NotificationAction(id: 'open_log',    label: 'Log',             params: {definitionId, windowDate}),
    NotificationAction(id: 'grant_grace', label: '15 more minutes', params: {definitionId, windowDate}),
    NotificationAction(id: 'dismiss',     label: 'Dismiss',         params: {}),
  ]
)
```

The `grant_grace` action carries `definitionId` and `windowDate` in its params. `GraceService` subscribes to `NotificationActionEvent` where `actionId == 'grant_grace'` and handles the rest. This service never calls `GraceService` directly — Grace sits above Commitment in the chain.

`log_done` and `open_log` actions are handled by subscribers to `NotificationActionEvent` within the Commitment feature.

---

## Notification Formatting

Warning body time remaining:

- > 60 min → hours, rounded to nearest half hour. Example: "about 2 hours left"
    
- 15–60 min → minutes, rounded to nearest 5. Example: "30 minutes left"
- < 15 min → no warning sent (`AppConfig.minimumWarningLeadMinutes`)

All formatting via `TemporalHelper.formatDuration()`.

---

## Notification ID Convention

IDs are stable and deterministic: `'{type}_{definitionId}_{windowStart}'`. This ensures `NotificationService.schedule()` replaces rather than duplicates when a window changes, and `NotificationService.cancel()` can target the correct notification without storing IDs locally.

---

## Rules

- Every scheduling decision passes through `_shouldNotify(instance)` first
- Calls `NotificationService.schedule()` and `NotificationService.cancel()` — never the OS directly
- Grace button carries `actionId: 'grant_grace'` in notification payload — never calls `GraceService` directly
- No scoring logic, no instance state changes
- Warning only scheduled if `instance.activityWindow.warningEnabled == true`
- Notification IDs are deterministic — no local ID storage needed
- All time formatting via `TemporalHelper.formatDuration()`

---

## Dependencies

- `EventBus` — subscribes to `InstanceCreatedEvent`, `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `CommitmentIdentityService` — reads instance for scheduling decisions
- `NotificationService` — schedules and cancels notifications
- `TemporalHelper` — time formatting
- `AppConfig` — `minimumWarningLeadMinutes`