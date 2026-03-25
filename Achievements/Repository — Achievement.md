**File Name**: repository_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage for `AchievementRecord`. The unified achievement history. Called only by `AchievementService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveRecord(record)`|AchievementRecord|—|
|`getAchievements(limit?, type?)`|optional filters|List ordered by earnedAt desc|
|`watchRecentAchievements(limit?)`|optional limit|Stream<List<AchievementRecord>>|
|`existsForSource(sourceId)`|sourceId|bool — idempotency check|

---

## Firestore Path

```
/users/{userId}/achievements/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `AchievementService`
- Append-only — no update or delete operations except full account deletion
- `existsForSource` used before every write for idempotency