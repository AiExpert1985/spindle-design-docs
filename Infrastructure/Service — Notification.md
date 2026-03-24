**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Infrastructure **Phase**: 1

**Purpose:** single entry point for all user-facing notifications. Subscribes to events from across the app and decides what notifications to send. No feature calls `NotificationService` directly — it is purely reactive.

---

## Design Change From Original

Previously features called `NotificationService` directly to send notifications. This created coupling — every feature that needed to notify the user had to know about and depend on `NotificationService`.

The new design: `NotificationService` subscribes to events. When a relevant event occurs, it decides whether and how to notify the user. Features know nothing about notifications — they just publish events about what happened.

---

## Events Subscribed

|Event|Notification sent|
|---|---|
|`WindowWarningDueEvent`|"Morning Walk window closes in 1 hour" + action buttons|
|`WindowClosedWithNoLogEvent`|"Morning Walk window just closed. Did you do it?" + action buttons|
|`StreakMilestoneReachedEvent`|"7 days in a row — Gold streak." (thin)|
|`CupEarnedEvent`|"You earned a silver cup this week." (thin)|
|`LevelReachedEvent`|Level-specific message from `ProgressionService.getLevelUpMessage()` (thin)|
|`DayCelebrationSignal`|"Your rope grew stronger today. 83%" (thin) — Path B only|

For `LevelReachedEvent`: reads the copy from the event payload — `ProgressionService` puts the message string into the event so `NotificationService` does not need to call `ProgressionService`.

---

## Two Notification Types

### Thin — awareness only

The notification informs. Tapping opens the relevant screen. No action buttons.

### Rich — actions inside the notification

Contains action buttons. User responds without opening the app. Used only for simple binary decisions.

Examples:

- Window closed check-in → `[ Yes ✓ ] [ Give me 15 min ] [ Skip ]`
- Coffee limit exceeded → `[ Log it ] [ Add note ]`

---

## Rule: Input Fields Stay in the App

Notifications never contain text input or quantity fields. Any interaction requiring input opens the app instead.

---

## Notification Categories

|Category|Type|Deep-link|
|---|---|---|
|Window warning|Rich|Dashboard|
|Window closed check-in|Rich|—|
|Grace follow-up|Rich|—|
|Avoid limit exceeded|Rich|—|
|Streak milestone|Thin|`/record?section=streaks`|
|Cup earned|Thin|`/record?section=cups`|
|Level up|Thin|`/progression`|
|Weekly report ready|Thin|Reports screen|
|Day celebration|Thin|Day celebration screen|

---

## Abstraction

```
NotificationService (abstract)
  → LocalNotificationService    (flutter_local_notifications)
  → (future: FCM for Pro/Premium push notifications)
```

The rest of the app never knows which implementation is active.

---

## Rules

- Subscribes to events — never called directly by any feature
- Reads notification copy from event payloads where possible — avoids calling other services
- Thin notifications for awareness and complex interactions
- Rich notifications for simple binary or single-tap actions only
- No input fields inside notifications
- Respects global notification settings from `UserService.getPreferences()`

---

## Dependencies

- `EventBus` — subscribes to all relevant events
- `UserService` — reads notification preferences
- Platform notification API (via abstraction)