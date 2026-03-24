**File Name**: component_window_warning **Feature**: Infrastructure **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** reminds the user before a commitment window closes, giving them time to act. A notification-driven flow — no in-app UI. Documents the complete warning experience across all involved services.

Standalone — can be disabled per commitment via `activityWindow.warningEnabled`. Removing it requires no changes to any other feature.

---

## Two Warning Paths

### Path 1 — Scheduled Warning (Do Commitments)

Fires when 3/4 of the activity window has elapsed and the commitment is not yet complete.

`NotificationSchedulerService` evaluates this on every `ShortIntervalTickEvent` for each active pending instance where `activityWindow.warningEnabled == true`. Idempotency enforced by `NotificationTrackingService` — fires once per instance only.

**Notification:**

```
"Morning Walk — 1 hour left in your window"
[ Log ]  [ Dismiss ]
```

"Log" opens `component_log_entry_dialog` with `dateLocked: true`. "Dismiss" takes no action.

---

### Path 2 — Immediate Breach Notification (Avoid Commitments)

Fires immediately when an avoid commitment limit is exceeded. `NotificationSchedulerService` subscribes to `ActivityRecordedEvent` and checks whether the log caused a breach — if yes, sends notification immediately.

**Notification:**

```
"Coffee limit exceeded"
[ Add note ]  [ Dismiss ]
```

"Add note" opens `component_log_entry_dialog` for the breaching log entry. "Dismiss" takes no action.

---

## Rules

- Only fires if `activityWindow.warningEnabled == true`
- Never fires for frozen or completed commitments
- Path 1 fires once per instance — enforced by `NotificationTrackingService`
- Path 2 fires once per breaching log entry — enforced by `NotificationTrackingService`
- Minimum lead time respected — warning skipped if less than `AppConfig.minimumWarningLeadMinutes` remains

---

## Components and Services Involved

|Piece|Role|
|---|---|
|`NotificationSchedulerService`|Evaluates warning conditions, schedules and sends notifications|
|`NotificationTrackingService`|Idempotency — ensures each notification fires once only|
|`component_log_entry_dialog`|UI shown when user taps Log from the notification|
|`ActivityService`|Records the log entry when user acts|
|`AppConfig.minimumWarningLeadMinutes`|Minimum useful warning lead time|