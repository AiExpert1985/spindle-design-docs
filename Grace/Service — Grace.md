**File Name**: graceservice **Feature**: Grace **Phase**: 2 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** gives the user a short extra window to log after their activity window closes. Purely additive — hooks into the notification system from above without modifying any lower feature. Fully removable: deleting this feature requires no changes to any other feature's code.

Grace is user-initiated. The user taps "Give me 15 minutes" on the window-close notification. A `NotificationActionEvent` with `actionId: 'grant_grace'` arrives. `GraceService` schedules a follow-up notification via `NotificationService`. If the user logs during the grace window, `ActivityService` records it normally — no special grace handling needed anywhere downstream.

---

## Why Grace Is Its Own Feature

Grace is optional, additive, and Phase 2. It sits above Commitment and Notification in the chain — it reads grace preferences from `UserCoreService`, schedules notifications through `NotificationService`, and subscribes to `InstancePermanentlyDeletedEvent` for cleanup. No lower feature knows Grace exists. Removing Grace in a future version requires zero changes below it.

---

## Feature Position

Grace sits at the same level as Streak, Cups, Rewards, and Analytics — above Performance, independent of those same-level features. It depends only on Commitment (for `InstancePermanentlyDeletedEvent`) and Notification (for `NotificationService`).

---

## GraceRecord Model

```
GraceRecord
  id: String
  definitionId: String
  windowDate: Date
  graceUntil: DateTime
  fired: bool
  createdAt: DateTime
```

- **definitionId + windowDate** — identifies which commitment window this grace belongs to. No `instanceId` — consistent with `LogEntry` design.
- **graceUntil** — when the grace window expires.
- **fired** — true once the follow-up notification has been scheduled. Ensures exactly one follow-up per grace grant.

---

## Events Subscribed

### `NotificationActionEvent` where `actionId == 'grant_grace'` → `_onGraceRequested(event)`

```
definitionId = event.params['definitionId']
windowDate   = Date.parse(event.params['windowDate'])

grantGrace(definitionId, windowDate)
```

The commitment window-close notification carries `action: 'grant_grace'` with `definitionId` and `windowDate` in its params. `GraceService` reacts to this event — Commitment never calls Grace directly.

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Deletes all `GraceRecord` entries for `event.definitionId`. Keeps the repository clean when a commitment is permanently removed.

---

## Functions

### `grantGrace(definitionId, windowDate)`

```
if GraceRepository.getGraceRecord(definitionId, windowDate) != null:
  return   // already granted — idempotent

prefs = UserCoreService.getGracePreferences()
if not prefs.graceEnabled or prefs.gracePeriodMinutes == 0:
  return   // grace disabled for this user

graceUntil = now + Duration(minutes: prefs.gracePeriodMinutes)

record = GraceRecord(
  definitionId: definitionId,
  windowDate: windowDate,
  graceUntil: graceUntil,
  fired: false
)
GraceRepository.saveGraceRecord(record)

NotificationService.schedule(
  ThinNotification(
    id: 'grace_followup_${definitionId}_${windowDate}',
    title: commitmentName,
    body: 'Your extra time is up.',
    deepLink: '/log?definitionId=${definitionId}'
  ),
  fireAt: graceUntil
)

GraceRepository.markFired(record.id)
```

One grace per window — `grantGrace()` is idempotent. After the follow-up notification fires, the user either logs (via the deep-link or the app) or does not. Either way, `ActivityService` and `PerformanceService` handle the result normally.

---

## Rules

- One grace per window — `grantGrace()` is idempotent
- Never calls `ActivityService`, `CommitmentIdentityService`, or `PerformanceService`
- Never modifies instance state or `livePerformance`
- Reacts to the grace button via `NotificationActionEvent` — never called directly by Commitment
- Grace preferences read from `UserCoreService` — never hardcoded
- Follow-up notification scheduled through `NotificationService` — GraceService never calls the OS directly

---

## Dependencies

- `GraceRepository` — reads and writes `GraceRecord`
- `NotificationService` — schedules follow-up notification
- `UserCoreService` — reads `graceEnabled` and `gracePeriodMinutes`
- `EventBus` — subscribes to `NotificationActionEvent`, `InstancePermanentlyDeletedEvent`
- `AppConfig` — `gracePeriodMinutes` as fallback default