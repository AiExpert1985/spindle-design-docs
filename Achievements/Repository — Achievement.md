**File Name**: repository_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage for `AchievementRecord`. The unified achievement history. Called only by `AchievementService`.

---

## Interface

```dart
abstract class AchievementRepository {
  Future<void> saveRecord(AchievementRecord record);
  Future<bool> existsForSource(String sourceId);
  Future<List<AchievementRecord>> getAchievements(DateTime from, DateTime to, {AchievementType? type});
  Stream<List<AchievementRecord>> watchRecentAchievements(DateTime from, DateTime to);
}
```

`saveRecord` — append-only. Never called for updates.

`existsForSource(sourceId)` — checks whether a record already exists for this source model. Used as a defensive safety net before every write. Producing features (Cups, Rewards, Milestones) guarantee no duplicate events through their own idempotency — `existsForSource` is a secondary guard that prevents corruption if a duplicate somehow arrives despite those guarantees.

`getAchievements(from, to, type?)` — all achievement records within the date range, optionally filtered by type. Ordered by `createdAt` descending. Used by the achievements screen.

`watchRecentAchievements(from, to)` — live stream of achievements within the date range. Used by the achievements screen for live updates when a new achievement is earned during the session.

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
- Achievement records are permanent — never deleted, not even on commitment deletion
- `existsForSource` called before every write as a defensive safety net
- All reads use a time window — never fetch unbounded history