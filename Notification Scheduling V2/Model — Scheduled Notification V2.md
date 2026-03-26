**File Name**: model_scheduled_notification **Feature**: Notification Scheduling **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** represents a single pending notification waiting to be fired. Owned by the Notification Scheduling feature. Created when a commitment window is scheduled, deleted when the notification fires or the commitment changes.

---

## Fields

```
ScheduledNotification
  id:                    String
  definitionId:          String
  notificationTimestamp: DateTime
  message:               String
  route:                 String?
  createdAt:             DateTime
  updatedAt:             DateTime
```

`id` — deterministic, constructed from `definitionId`, window date, and notification type (e.g. `'warning_abc123_2026-03-26'`). Because IDs are stable, saving a record with the same ID replaces the previous one — no duplicate records on reschedule.

`definitionId` — links this record to its commitment. Used to clear all pending notifications when the commitment changes.

`notificationTimestamp` — when this notification is due to fire. The service checks this against the current time on each Heartbeat tick.

`message` — the text shown to the user. Constructed at write time — the service that delivers the notification uses it as-is.

`route` — optional GoRouter path. When the user taps the notification, the app navigates here. If null, the app opens at the home screen.

---

## Rules

- `id` must be deterministic — never random. Stable IDs make reschedule a replace, not an insert.
- `message` is final at write time — never modified after creation
- A record's presence means the notification is pending. Its absence means it fired or was cancelled — no status field needed.