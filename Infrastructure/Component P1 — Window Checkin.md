**File Name**: component_window_checkin **Feature**: Infrastructure **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** when a do commitment window closes with no log, asks the user whether they did it rather than silently marking it missed. Treats the user as someone who may have forgotten — not someone who failed. A notification-driven flow — no in-app UI.

Standalone — removing it requires no changes to any other feature. Applies to do commitments only — avoid commitments breach immediately on log, no check-in needed.

---

## The Flow

### Step 1 — Window Close Notification

`NotificationSchedulerService` detects a closed instance with no log entry via `InstanceUpdatedEvent` where `instance.status == closed`. Fires the check-in notification if the instance has no log and `activityWindow.warningEnabled == true`. Idempotency enforced by `NotificationTrackingService`.

**Binary commitment:**

```
"Morning Walk — did you do it?"
[ Done ]  [ 15 more minutes ]  [ Dismiss ]
```

"Done" calls `ActivityService.recordEntry(value: 1, loggedAt: windowDate)` directly — no UI shown.

**Numerical commitment:**

```
"Morning Walk — did you do it?"
[ Log ]  [ 15 more minutes ]  [ Dismiss ]
```

"Log" opens `component_log_entry_dialog` with `dateLocked: true` and `date: windowDate`.

"15 more minutes" calls `GraceService.grantGrace(definitionId, windowDate)` — schedules a follow-up notification after the grace period. "Dismiss" takes no action — instance remains with whatever `livePerformance` it had.

---

### Step 2 — Grace Follow-Up (if requested)

If the user tapped "15 more minutes", `GraceService` fires a follow-up notification after `gracePeriodMinutes` (user-configurable, default 15 min). One grace per window — cannot be requested twice.

**Follow-up notification:**

```
"Morning Walk — last chance"
[ Done ] / [ Log ]  [ Dismiss ]
```

Same action buttons as Step 1. "Dismiss" is final — no further follow-up.

---

## Rules

- Applies to do commitments only — never fires for avoid commitments
- Never fires for frozen or completed commitments
- Fires once per instance — enforced by `NotificationTrackingService`
- One grace per instance — `GraceService.grantGrace()` is idempotent
- Dismiss is always silent — no missed log written, no penalty

---

## Components and Services Involved

|Piece|Role|
|---|---|
|`NotificationSchedulerService`|Detects closed instance with no log, sends check-in notification|
|`NotificationTrackingService`|Idempotency — fires once per instance only|
|`GraceService`|Schedules and fires the grace follow-up notification|
|`component_log_entry_dialog`|UI shown when user taps Log (numerical commitments)|
|`ActivityService.recordEntry()`|Records the log when user confirms Done or Log|