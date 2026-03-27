**File Name**: notification_service **Feature**: Infrastructure **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the single delivery path for all OS notifications. Accepts a message and an optional route from any feature, and delivers the notification to the user's device. Has no knowledge of why a notification was requested, what it means, or who sent it.

---

## Design

`NotificationService` is an abstract interface. The rest of the app depends only on the interface — never on a concrete implementation. This means the underlying delivery package can be swapped or upgraded without touching any feature.

```dart
abstract class NotificationService {
  Future<void> push({
    required String message,
    String? route,
  });
}
```

`message` — the text shown to the user in the notification tray.

`route` — optional GoRouter path. When the user taps the notification, the app navigates to this route (e.g. `/log?definitionId=abc123`). If null, the app opens at the home screen.

---

## Why Notifications Are Thin

Notifications carry only a message and a route. No action buttons, no rich payloads, no params maps.

Rich notifications — where the user taps an action button inside the notification itself — require background execution, service callbacks, and result sync. That complexity is not justified for Phase 1. Any interaction beyond acknowledging the notification opens the app, where the full UI is available.

If richer interactions become necessary in a later phase, the interface is extended at that point — nothing in the current design prevents it.

---

## Implementations

```
NotificationService (abstract interface)
  └── LocalNotificationService   ← Phase 1, flutter_local_notifications
  └── (future: remote push)      ← Phase 3, FCM for Pro/Premium
```

One implementation is active at a time, injected via dependency injection.

---

## Rules

- Features never call the OS notification API directly — always through this interface
- The caller constructs the message — `NotificationService` never generates notification text
- If OS notification permission is denied, delivery is silently dropped and logged via `LoggerService`
- `NotificationService` has no knowledge of any feature, model, or domain concept