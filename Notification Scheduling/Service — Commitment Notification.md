**File Name**: service_commitment_notifications **Feature**: CommitmentNotifications **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** maintains pending notification records for commitment windows and fires them at the right time. Watches instance events to keep records current. Watches Heartbeat to fire due records. Has no knowledge of scoring, performance, or any other domain concept.

---

## Watching Instances

Subscribes to `InstanceCreatedEvent`, `InstanceUpdatedEvent`, and `InstancePermanentlyDeletedEvent` from `CommitmentIdentityService`.

**On `InstanceCreatedEvent` or `InstanceUpdatedEvent`:**

1. Delete all existing records for that `definitionId`
2. Re-evaluate the instance
3. If `commitmentState == active` and the window has not yet passed, write fresh records

**On `InstancePermanentlyDeletedEvent`:**

Delete all records for that `definitionId`. No re-evaluation.

---

## Records Written Per Instance

### Warning notification

Fires before the window closes, giving the user time to act. Scheduled at 3/4 of the window duration before window end.

Skipped if:

- `activityWindow.warningEnabled == false`
- Lead time is less than `AppConfig.minimumWarningLeadMinutes` — too short to be useful

```
message: "{commitmentName} — your window closes in {formatted duration}"
route:   "/log?definitionId={definitionId}"
id:      "warning_{definitionId}_{windowDate}"
```

Warning lead time formatted via `TemporalHelper`:

- > 60 min → rounded to nearest half hour: "about 2 hours"
    
- 15–60 min → rounded to nearest 5 minutes: "30 minutes"
    
- < `AppConfig.minimumWarningLeadMinutes` → record not written
    

### Checkin notification

Fires at window end when the user has not yet logged. Asks the user if they did it.

```
message: "{commitmentName} — did you do it?"
route:   "/log?definitionId={definitionId}"
id:      "checkin_{definitionId}_{windowDate}"
```

Both records are written only when `commitmentState == active`.

---

## Watching Heartbeat

Subscribes to `Heartbeat.shortIntervalTick`. On each tick:

1. Query `getDue(now)` for all records with `notificationTimestamp <= now`
2. For each due record: call `NotificationService.push(message, route)`, then delete the record

---

## Dependencies

- `CommitmentIdentityService` — subscribes to `InstanceCreatedEvent`, `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`
- `ScheduledNotificationRepository` — reads and writes pending records
- `Heartbeat` — subscribes to `shortIntervalTick`
- `NotificationService` — delivers fired notifications
- `TemporalHelper` — formats warning lead time for message text
- `AppConfig` — `minimumWarningLeadMinutes`

---

## Rules

- Warning notification skipped if `activityWindow.warningEnabled == false` or lead time is below `AppConfig.minimumWarningLeadMinutes`
- Both notification records written only when `commitmentState == active`
- All functions return `Result<T>` — no raw exceptions

---

## Later Improvements

**Grace.** Grace gives the user a short extra window to log after their commitment window closes. The user triggers it from the activity log screen ("give me 15 more minutes"). This writes a grace record with `definitionId` and a `graceUntil` timestamp — the same model and repo this feature already uses. The Heartbeat watch fires a follow-up checkin notification when `graceUntil` is reached, then deletes the record.

Grace is a natural extension of this feature: same mechanism, same repo, one additional write path triggered by user action rather than instance state. It is deferred because the UI trigger — a button in the activity log — requires design decisions not yet made for that screen.