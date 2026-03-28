**File Name**: repository_commitment_notifications **Feature**: CommitmentNotifications **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** stores and retrieves pending `ScheduledNotification` records. Called only by `NotificationSchedulingService`. No business logic — only fetch and store.

---

## Interface

```dart
abstract class ScheduledNotificationRepository {
  Future<void> save(ScheduledNotification notification);
  Future<List<ScheduledNotification>> getByDefinitionId(String definitionId);
  Future<List<ScheduledNotification>> getDue(DateTime now);
  Future<void> delete(String id);
  Future<void> deleteByDefinitionId(String definitionId);
}
```

`save(notification)` — inserts or replaces by `id`. Because IDs are deterministic, calling save with the same ID is a reschedule, not a duplicate.

`getByDefinitionId(definitionId)` — returns all pending records for one commitment. Used to verify state before re-evaluating.

`getDue(now)` — returns all records where `notificationTimestamp <= now`. Called on every Heartbeat tick to find what needs to fire.

`delete(id)` — removes one record. Called immediately after a notification fires.

`deleteByDefinitionId(definitionId)` — removes all records for one commitment. Called when an instance changes or is deleted, before re-evaluation.

---

## Rules

- Called only by `NotificationSchedulingService` — no other feature accesses this repository
- No business logic — fetch and store only
- `getDue` uses a single range filter on `notificationTimestamp` — compatible with Firestore query constraints