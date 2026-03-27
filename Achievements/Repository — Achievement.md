**File Name**: repository_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage for `AchievementRecord`. The single source of truth for all achievement history. Called only by `AchievementService`.

---

## Interface

```dart
abstract class AchievementRepository {
  Future<void> saveRecord(AchievementRecord record);
  Future<bool> existsForSource(String sourceId);
  Future<List<AchievementRecord>> getAchievements(
    DateTime from,
    DateTime to, {
    AchievementType? type,
    AchievementSubtype? subtype,
    String? definitionId,
  });
  Stream<List<AchievementRecord>> watchAchievements(DateTime from, DateTime to);
}
```

`saveRecord` — append-only. Never called for updates.

`existsForSource(sourceId)` — idempotency safety net. Called before every write. Producing features guarantee no duplicate calls — this is a secondary defensive check.

`getAchievements(from, to, type?, subtype?, definitionId?)` — unified read. All parameters optional except the time window. Ordered by `createdAt` descending. Used for all reads — cups history, streak achievements, garment achievements, all of them.

`watchAchievements(from, to)` — live stream. Used by the achievements screen during a session.

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
- Records are permanent — never deleted for any reason
- `existsForSource` called before every write
- All reads use a time window — never fetch unbounded history