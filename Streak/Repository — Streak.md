**File Name**: repository_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** storage interface for the Streak feature. Stores one `StreakRecord` per commitment. Called only by `StreakService`.

---

## Interface

```dart
abstract class StreakRepository {
  Future<void> saveRecord(StreakRecord record);
  Future<StreakRecord?> getRecord(String definitionId);
  Stream<StreakRecord?> watchRecord(String definitionId);
  Future<void> deleteRecord(String definitionId);
  Future<List<StreakRecord>> getAllRecords();
}
```

`saveRecord` — upsert. Called after every streak update.

`getRecord` — returns null if no record exists yet. `StreakService` creates it lazily on first window evaluation.

`watchRecord` — live stream for UI display.

`deleteRecord` — hard delete. Called only on `InstancePermanentlyDeletedEvent` cascade.

`getAllRecords` — returns all streak records for the current user. Used by `_findBestStreak()` to compute the global best on demand. At most a few dozen records per user.

---

## Firestore Path

```
/users/{userId}/streakRecords/{definitionId}
```

`definitionId` is the document ID — no separate id field. Covered by the existing security rule.

---

## Rules

- Called only by `StreakService`
- No business logic — only fetch and store
- `deleteRecord` called only on `InstancePermanentlyDeletedEvent` cascade