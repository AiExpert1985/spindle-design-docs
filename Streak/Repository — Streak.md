**File Name**: repository_streak **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage interface for the Streak feature. Stores `StreakRecord` per commitment and one `GlobalBestStreakRecord` per user. Called only by `StreakService`.

---

## StreakRecord Operations

```dart
abstract class StreakRepository {
  Future<void> saveRecord(StreakRecord record);
  Future<StreakRecord?> getRecord(String definitionId);
  Stream<StreakRecord?> watchRecord(String definitionId);
  Future<void> deleteRecord(String definitionId);
  Future<void> saveGlobalBest(GlobalBestStreakRecord record);
  Future<GlobalBestStreakRecord?> getGlobalBest();
}
```

`saveRecord` — upsert. Called after every streak update.

`getRecord` — returns null if no record exists yet. `StreakService` creates it lazily on first window evaluation for this commitment.

`watchRecord` — live stream. Used by the commitment detail screen for live streak display. Detached when the screen is no longer visible.

`deleteRecord` — hard delete. Called only on `InstancePermanentlyDeletedEvent` cascade.

`saveGlobalBest` — upsert. Called whenever a new all-time best is set.

`getGlobalBest` — returns the single `GlobalBestStreakRecord`, or null if no best has been set yet. Single-document read — no collection scan.

---

## Firestore Paths

```
/users/{userId}/streakRecords/{definitionId}    ← one per commitment
/users/{userId}/globalBestStreak/record         ← single document
```

Both covered by the existing security rule. `definitionId` is the document ID for streak records — no separate id field.

---

## Rules

- Called only by `StreakService`
- No business logic — only fetch and store
- `deleteRecord` called only on `InstancePermanentlyDeletedEvent` cascade
- `watchRecord` stream detached when the observing screen is no longer visible
- `getAllRecords` is not provided — global best is tracked explicitly via `GlobalBestStreakRecord`, not derived by scanning all records