**File Name**: component_window_warning **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** reminds the user before a commitment window closes, giving them time to act. A notification-driven flow — no in-app UI. Documents the complete warning experience across all involved services.

Standalone — can be disabled per commitment via `activityWindow.warningEnabled`. Removing it requires no changes to any other feature.

---

## Two Warning Paths

### Path 1 — Scheduled Warning (Do Commitments)

`CommitmentNotificationSchedulerService` schedules a warning notification via `NotificationService.schedule()` at 3/4 of the activity window duration, subject to `AppConfig.minimumWarningLeadMinutes`. Only scheduled if `activityWindow.warningEnabled == true` and `commitmentState == active`.

Idempotency guaranteed by stable notification ID — `NotificationService` replaces rather than duplicates when the instance is recreated.

**Notification:**

```
"Morning Walk — 1 hour left in your window"
[ Log ]  [ Dismiss ]
```

"Log" carries `actionId: 'open_log'` — opens `component_log_entry_dialog` with `dateLocked: true`. "Dismiss" takes no action.

---

### Path 2 — Immediate Breach Notification (Avoid Commitments)

`CommitmentNotificationSchedulerService` subscribes to `ActivityEvent` (created only) and checks whether the new log caused an avoid commitment to exceed its target. If so, schedules an immediate notification via `NotificationService.schedule(payload, fireAt: now)`.

Idempotency by stable notification ID incorporating `loggedAt` — one breach notification per log entry.

**Notification:**

```
"Coffee limit exceeded"
[ Add note ]  [ Dismiss ]
```

"Add note" carries `actionId: 'open_log'` — opens `component_log_entry_dialog` for the breaching log entry. "Dismiss" takes no action.

---

## Rules

- Path 1 only fires if `activityWindow.warningEnabled == true`
- Neither path fires for frozen or completed commitments — `_shouldNotify()` gate in `CommitmentNotificationSchedulerService`
- Idempotency via stable notification IDs — no separate tracking service needed
- Minimum lead time respected on Path 1 — warning skipped if less than `AppConfig.minimumWarningLeadMinutes` remains
- Path 2 evaluation is a pure check on `livePerformance` after `ActivityEvent` — no repository read needed

---

## Services Involved

|Piece|Role|
|---|---|
|`CommitmentNotificationSchedulerService`|Schedules both warning types via `NotificationService`|
|`NotificationService`|Delivers notifications, publishes `NotificationActionEvent` on action tap|
|`component_log_entry_dialog`|UI shown when user taps Log|
|`ActivityService`|Records the log entry when user acts|
|`AppConfig.minimumWarningLeadMinutes`|Minimum useful warning lead time|