**File Name**: graceservice **Feature**: Infrastructure / Notification **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** gives the user a short extra window to log after their activity window closes. A dumb timer — registers a delay and re-fires the log notification once. Knows nothing about instances, performance, or scoring.

---

## Design

Grace is user-initiated. The user taps "Give me 15 minutes" in the window-close notification. A timer record is created. When it expires, the same notification fires again. If the user logs, ActivityService records it normally. PerformanceService finds the correct instance by date and updates livePerformance — no special grace handling needed anywhere downstream.

If the user dismisses the follow-up, nothing happens. The instance closes with whatever livePerformance it had. Silence is a valid outcome.

Grace is capped at one occurrence per instance and is user-initiated — it cannot be triggered automatically. The duration is user-configurable. Setting it to zero disables grace.

Grace is in the Infrastructure/Notification folder because it is purely a notification re-scheduling mechanism with no domain knowledge. Any feature that needs "re-notify after delay" could use the same pattern.

---

## Grace Record

```
GraceRecord
  id: String
  definitionId: String
  windowDate: Date
  graceUntil: DateTime
  fired: bool
  createdAt: DateTime
```

- **definitionId + windowDate** — identifies which commitment window this grace belongs to. Used to find the record and to clean up on permanent deletion.
- **graceUntil** — when the timer expires.
- **fired** — true once the follow-up notification has been sent. Ensures the notification fires exactly once.

No `instanceId` — consistent with LogEntry design. The window is identified by commitment and date.

---

## Events Subscribed

### `ShortIntervalTickEvent`

Checks all `GraceRecord` entries where `graceUntil <= now` and `fired == false`. For each: sends follow-up notification via NotificationService, sets `fired: true`. Uses TickGuard to prevent double-processing.

### `CommitmentUpdatedEvent` where `changes` contains `permanentlyDeleted`

Deletes all GraceRecords for this `definitionId`.

---

## Functions

### `grantGrace(definitionId, windowDate)`

Called when user taps the grace button.

1. Check if GraceRecord already exists for this `definitionId + windowDate` — exit silently if yes. One grace per window.
2. Read duration from user settings (`gracePeriodMinutes`), fallback to `AppConfig.gracePeriodMinutes`.
3. Create `GraceRecord(graceUntil: now + duration, fired: false)`.

### `_sendFollowUpNotification(definitionId, windowDate)`

Called internally when grace expires. Sends notification with inline log action via NotificationService. Sets `fired: true`.

The notification action calls `ActivityService.recordEntry(definitionId, ...)` — same path as any manual log. GraceService is not involved after the notification fires.

---

## Rules

- One grace per window — `grantGrace()` is idempotent
- Follow-up fires exactly once per expired grace — `fired` flag enforces this
- Never calls ActivityService, CommitmentIdentityService, or PerformanceService
- Never modifies instance state or livePerformance directly

---

## Dependencies

- GraceRepository — reads and writes GraceRecord
- NotificationService — sends follow-up notification
- EventBus — subscribes to tick and CommitmentUpdatedEvent
- TickGuard — prevents double-processing
- AppConfig — gracePeriodMinutes default
- UserSettings — per-user grace duration override