**File Name**: notificationschedulerservice **Feature**: Infrastructure **Phase**: 1 **Created**: 20-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** owns all notification timing decisions. Schedules and cancels notifications in response to instance lifecycle events. The single location for "when should a notification fire" logic. Does not send notifications directly — delegates to NotificationService.

---

## The Notification Gate

Every scheduling decision passes through `_shouldNotify(instance)` before any notification is scheduled or sent. This function is the single enforcement point for notification eligibility.

```
_shouldNotify(instance) → bool
  returns true only if instance.commitmentState == active
```

Frozen, completed, and soft-deleted commitments do not receive notifications. This check is applied on every `InstanceUpdatedEvent` and before every scheduling decision — not just at creation time. If a commitment transitions to non-active mid-schedule, the next tick or update cancels any pending notifications.

---

## Events Subscribed

### `InstanceCreatedEvent` → `_onInstanceCreated(event)`

New instance created. Reads instance via `CommitmentIdentityService.getCurrentInstance(definitionId)`. Calls `_shouldNotify(instance)` — if false, exits. If true, schedules warning and window-close notifications based on `instance.activityWindow`.

### `InstanceUpdatedEvent` → `_onInstanceUpdated(event)`

Instance changed. Reads updated instance. Calls `_shouldNotify(instance)`:

- If false — cancels all pending notifications for this instance.
- If true — reschedules notifications from the current instance state. Handles activity window changes and any other structural updates in one pass.

### `InstancePermanentlyDeletedEvent` → `_onInstanceDeleted(event)`

Cancels all pending notifications for this `definitionId`. Permanent — no reschedule.

---

## Notification Types

### Warning notification

Fires at 3/4 of the `activityWindow` duration, subject to minimum lead time.

- Time remaining > `AppConfig.minimumWarningLeadMinutes` → schedule warning
- Time remaining ≤ `AppConfig.minimumWarningLeadMinutes` → skip — not enough time to be useful

Content: "Morning Run — 30 minutes left in your window." Time formatting via `TemporalHelper.formatDuration()` — never hardcoded.

Only sent if `instance.activityWindow.warningEnabled == true`.

### Window-close notification

Fires at `activityWindow.startMinutes + activityWindow.durationMinutes`.

Content varies by `commitmentType`:

**Binary:**

```
"Morning Run — time is up"
[ Done ]  [ 15 more minutes ]  [ Dismiss ]
```

**Numerical:**

```
"Morning Run — time is up"
[ Log ]  [ 15 more minutes ]  [ Dismiss ]
```

"Done" (binary) calls `ActivityService.recordEntry(value: 1, loggedAt: windowDate)` directly. "Log" (numerical) opens `LogEntryDialog` with `dateLocked: true`. "15 more minutes" calls `GraceService.grantGrace(definitionId, windowDate)`. "Dismiss" takes no action.

---

## Notification Formatting

Time remaining in warning notifications:

- > 60 min → hours, rounded to nearest half hour. Example: "about 2 hours left"
    
- 15–60 min → minutes, rounded to nearest 5. Example: "30 minutes left"
- < 15 min → no warning sent

All formatting via `TemporalHelper.formatDuration()`.

---

## Rules

- Every scheduling decision passes through `_shouldNotify(instance)` first
- Schedules and cancels notifications only — never sends directly
- All sends go through NotificationService
- Idempotency for every notification type checked via NotificationTrackingService before sending
- No scoring logic, no instance state changes
- Warning only sent if `instance.activityWindow.warningEnabled == true`
- Notification content strings from the localization system — never hardcoded
- `AppConfig.minimumWarningLeadMinutes` governs minimum useful warning time

---

## Dependencies

- EventBus — subscribes to InstanceCreatedEvent, InstanceUpdatedEvent, InstancePermanentlyDeletedEvent
- CommitmentIdentityService — reads instance for scheduling decisions
- NotificationService — sends notifications
- NotificationTrackingService — idempotency tracking
- GraceService — grace period creation on notification action
- TemporalHelper — time formatting
- AppConfig — minimumWarningLeadMinutes