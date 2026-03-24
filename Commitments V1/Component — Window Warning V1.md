**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Commitment **Phase**: 1 **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** reminds the user before a commitment window closes, giving them time to act. Includes inline action buttons so the user can log without opening the app.

---

## Two Trigger Paths

**Path 1 — Scheduled warning (do commitments):** `CommitmentSchedulerService` detects 3/4 elapsed condition during the `WindowCheckEvent` cycle and publishes `WindowWarningDueEvent`. `NotificationService` subscribes and sends the rich notification.

**Path 2 — Immediate breach notification (avoid commitments):** `ActivityService` publishes `ActivityRecordedEvent` after writing a breach log entry. `NotificationService` subscribes — if `logResult` indicates an avoid breach, sends the rich notification immediately.

Both paths produce notifications. Neither path calls `NotificationService` directly — both work through events.

---

## Path 1 — Scheduled Window Warning (Do Commitments)

Fires when 3/4 of the commitment window has elapsed and commitment is not yet complete.

|Window duration|Fires at|Time remaining|
|---|---|---|
|4 hours|3h elapsed|1h left|
|1 day|18h elapsed|6h left|
|7 days|~5.25 days elapsed|~1.75 days left|

Custom override per commitment: `warningAmount` + `warningUnit` on the commitment definition.

**Notification format:**

```
"Morning Walk window closes in 1 hour"
[ Done ✓ ]  [ +0.5km ]  [ Snooze 30min ]
```

The `+0.5km` button uses `defaultLogIncrement` from the commitment definition. Inline action buttons call `ActivityService.markInstanceKept()` or `ActivityService.recordEntry()`.

---

## Path 2 — Immediate Breach Notification (Avoid Commitments)

Fires when an avoid commitment limit is exceeded. `NotificationService` detects this from `ActivityRecordedEvent` by checking the instance's avoid breach status.

**Notification format:**

```
"Coffee limit exceeded"
[ Log it ]  [ Add note ]
```

---

## Shared Rules

- Only fires if `warningEnabled: true` on the commitment definition
- Never fires for frozen commitments
- Never fires if commitment already complete for that window
- Path 1 tracks `notifiedWarning: bool` on the instance — fires once only per instance
- Time remaining shown as integers: nearest hour for hourly/daily windows, nearest day for weekly/custom

---

## Dependencies

- `CommitmentSchedulerService` — publishes `WindowWarningDueEvent`
- `ActivityService` — publishes `ActivityRecordedEvent` (breach detection)
- `NotificationService` — subscribes to both events, sends notifications
- `ActivityService` — inline action buttons call for quick-log and mark-kept