**File Name**: model_notification_payload **Feature**: Notification **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** describes a single notification to be delivered to the user. The typed payload that features construct and pass to `NotificationService`. The Notification feature knows nothing beyond what is in this model.

---

## Notification Variants

```
sealed class NotificationPayload
  ThinNotification
  RichNotification
```

**Thin** — awareness only. Tapping opens the app at a deep-link destination. No action buttons.

**Rich** — contains action buttons. User responds without opening the app. Used only for simple single-tap decisions. Input fields are never inside notifications — any interaction requiring input opens the app.

---

## Fields

```
ThinNotification
  id: String                      // unique, used for cancel and idempotency
  title: String
  body: String
  deepLink: String?               // GoRouter path, e.g. '/record?section=streaks'

RichNotification
  id: String
  title: String
  body: String
  deepLink: String?
  actions: List<NotificationAction>   // 1–3 actions

NotificationAction
  id: String                      // action identifier, e.g. 'grant_grace'
  label: String                   // displayed on the button
  params: Map<String, String>     // arbitrary key-value data passed with the action
```

---

## NotificationActionEvent

When a user taps an action button on a rich notification, `NotificationService` publishes:

```
NotificationActionEvent
  actionId: String          // matches NotificationAction.id
  params: Map<String, String>
```

Features that care about specific actions subscribe to this event and filter on `actionId`. The Notification feature never knows what any action means — it only fires the event.

---

## Rules

- `id` must be unique per notification context — callers are responsible for stable IDs (e.g. `'window_warning_${definitionId}_${windowStart}'`)
- `actions` list capped at 3 — OS limitation on most Android versions
- No input fields in notifications — any interaction requiring input must deep-link into the app
- `params` values are strings only — callers serialize/deserialize as needed
- Notification copy is always in the user's display language — callers handle localization before constructing the payload