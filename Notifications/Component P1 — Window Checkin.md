**File Name**: component_window_checkin **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** when a commitment window closes with no log, asks the user whether they did it rather than silently marking it missed. Treats the user as someone who may have forgotten — not someone who failed. A notification-driven flow — no in-app UI.

Standalone — removing it requires no changes to any other feature. Applies to do commitments only — avoid commitments breach immediately on log, no check-in needed.

---

## The Flow

### Step 1 — Window Close Notification

`CommitmentNotificationSchedulerService` schedules a window-close notification via `NotificationService.schedule()` when an instance is created. The notification fires at the end of the `activityWindow`. It is a `RichNotification` — see `Infrastructure___Notification_Schedulers` for the full payload definition.

Only fires if `commitmentState == active`. Idempotency guaranteed by stable notification IDs — `NotificationService` replaces rather than duplicates.

**Binary commitment:**

```
"Morning Walk — did you do it?"
[ Done ]  [ 15 more minutes ]  [ Dismiss ]
```

"Done" carries `actionId: 'log_done'` — handled by a `NotificationActionEvent` subscriber within the Commitment feature, which calls `ActivityService.recordEntry(value: 1, loggedAt: windowDate)`.

**Numerical commitment:**

```
"Morning Walk — did you do it?"
[ Log ]  [ 15 more minutes ]  [ Dismiss ]
```

"Log" carries `actionId: 'open_log'` — opens `component_log_entry_dialog` with `dateLocked: true` and `date: windowDate`.

"15 more minutes" carries `actionId: 'grant_grace'` — `GraceService` subscribes to `NotificationActionEvent` where `actionId == 'grant_grace'` and schedules the follow-up via `NotificationService.schedule()`. The Commitment feature never calls `GraceService` directly — Grace sits above Commitment in the chain.

"Dismiss" takes no action — instance remains with whatever `livePerformance` it had.

---

### Step 2 — Grace Follow-Up (if requested)

If the user tapped "15 more minutes", `GraceService` schedules a follow-up notification via `NotificationService`. Fires after `gracePeriodMinutes` (user-configurable via `UserCoreService.getGracePreferences()`). One grace per window — `GraceService.grantGrace()` is idempotent.

**Follow-up notification:**

```
"Morning Walk — last chance"
[ Done ] / [ Log ]  [ Dismiss ]
```

Same action buttons as Step 1. "Dismiss" is final — no further follow-up.

---

## Rules

- Applies to do commitments only — never fires for avoid commitments
- Never fires for frozen or completed commitments — `_shouldNotify()` gate in `CommitmentNotificationSchedulerService`
- Idempotency by stable notification ID — `NotificationService` enforces no duplicate delivery
- One grace per window — `GraceService.grantGrace()` is idempotent
- Dismiss is always silent — no missed log written, no penalty
- Grace action handled by `GraceService` via `NotificationActionEvent` — never called directly from Commitment

---

## Services Involved

|Piece|Role|
|---|---|
|`CommitmentNotificationSchedulerService`|Schedules window-close notification via `NotificationService`|
|`NotificationService`|Delivers notification, publishes `NotificationActionEvent` on action tap|
|`GraceService`|Subscribes to `NotificationActionEvent` where `actionId == 'grant_grace'`, schedules follow-up|
|`component_log_entry_dialog`|UI shown when user taps Log (numerical commitments)|
|`ActivityService.recordEntry()`|Records the log when user confirms Done or Log|