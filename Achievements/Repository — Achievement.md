**File Name**: repository_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** storage for `AchievementRecord`. Called only by `AchievementService`.

---

## Interface

```dart
abstract class AchievementRepository {
  Future<void> saveRecord(AchievementRecord record);
  Future<bool> existsForSource(String sourceId);
  Future<List<AchievementRecord>> getAchievements(
    DateTime from,
    DateTime to, {
    String? type,
    String? subtype,
    String? definitionId,
  });
  Stream<List<AchievementRecord>> watchAchievements(DateTime from, DateTime to);
}
```

`saveRecord` — append-only. Never called for updates.

`existsForSource(sourceId)` — idempotency safety net. Called before every write.

`getAchievements(from, to, type?, subtype?, definitionId?)` — unified read. All filters optional except the time window. Ordered by `createdAt` descending.

`watchAchievements(from, to)` — live stream.

---

## Firestore Path

```
/users/{userId}/achievements/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `AchievementService`
- Append-only — no update or delete operations
- `existsForSource` called before every write
- All reads use a time window — never fetch unbounded history