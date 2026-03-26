**File Name**: service_notification **Feature**: Notification **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the single delivery path for all user-facing notifications. Accepts notification payloads from any feature above it and delivers them immediately or at a scheduled time. Publishes `NotificationActionEvent` when a user taps an action button. Has no knowledge of what any notification means or why it was scheduled.

---

## Why This Is a Feature, Not Infrastructure

`NotificationService` subscribes to `ShortIntervalTickEvent` to evaluate due scheduled notifications. Subscribing to any event — even a tick — requires being a feature, not infrastructure. Infrastructure never subscribes to anything (see `architecture_rules` section 11).

Additionally, `NotificationService` owns a repository (`NotificationRepository`) for persisting scheduled notifications. Infrastructure has no persistent state of its own.

Any feature above it in the chain calls `NotificationService` as a normal downward service call.

---

## Public Interface

### `send(payload: NotificationPayload)`

Delivers a notification immediately via the OS notification API. Idempotent on `payload.id` — will not send the same notification ID twice within a session. For cross-session idempotency, callers use stable IDs (see `model_notification_payload`).

### `schedule(payload: NotificationPayload, fireAt: DateTime)`

Stores a `ScheduledNotification` record in `NotificationRepository`. The notification fires when `ShortIntervalTickEvent` arrives at or after `fireAt`. Replaces any existing scheduled notification with the same `payload.id`.

### `cancel(notificationId: String)`

Cancels a pending scheduled notification and removes its record. Also cancels any OS-level scheduled delivery for this ID. Safe to call when no notification exists for this ID — exits silently.

---

## Tick Subscription

### `ShortIntervalTickEvent` → `_onTick(event)`

```
due = NotificationRepository.getDueNotifications(now: event.timestamp)
for each record in due:
  send(record.payload)
  NotificationRepository.markFired(record.id)
```

TickGuard used per notification ID — prevents double-firing if two ticks arrive close together.

---

## Action Handling

When a user taps an action button on a rich notification, the OS calls back with the action ID and the notification payload params. `NotificationService` publishes:

```
NotificationActionEvent
  actionId: String
  params: Map<String, String>
```

The service never processes the action itself. Any feature that cares subscribes to `NotificationActionEvent` and filters on `actionId`.

---

## OS Abstraction

```
NotificationService (abstract)
  → LocalNotificationServiceImpl   (flutter_local_notifications — Phase 1)
  → (future: FCM for Pro/Premium push notifications — Phase 3)
```

The rest of the app never knows which implementation is active.

---

## Rules

- Feature-agnostic — never imports or subscribes to any other feature's events or models
- Callers construct notification copy — `NotificationService` never generates text
- Callers handle localization — payload text is always in the user's display language before calling
- `schedule()` replaces an existing record with the same ID — callers use stable IDs for rescheduling
- `NotificationActionEvent` is published for every action tap — action handling belongs in subscribers
- Respects OS-level notification permissions — silently drops delivery if permission denied, logs warning via `ErrorService`
- All OS calls go through the abstract interface — never call `flutter_local_notifications` directly outside the implementation class

---

## Dependencies

- `NotificationRepository` — stores scheduled notifications
- `EventBus` — subscribes to `ShortIntervalTickEvent`; publishes `NotificationActionEvent`
- `TickGuard` — per-notification firing idempotency
- `ErrorService` — logs delivery failures and permission denials
- OS notification API (via abstract interface)