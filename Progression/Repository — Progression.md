**File Name**: repository_progression **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage interface for the Progression feature. Stores `ProgressionProfile` and the `LevelRecord` collection. Called only by `ProgressionService`.

---

## Interface

```dart
abstract class ProgressionRepository {
  // ProgressionProfile
  Future<ProgressionProfile?> getProfile();
  Future<void> saveProfile(ProgressionProfile profile);
  Stream<ProgressionProfile?> watchProfile();

  // LevelRecord collection
  Future<List<LevelRecord>> getAllLevels();
  Future<LevelRecord?> getCurrentLevel();
  Future<void> saveLevel(LevelRecord record);
  Stream<List<LevelRecord>> watchAllLevels();
}
```

`getProfile` — returns null if no profile exists yet. Created on first `PointsAwardedEvent`.

`saveProfile` — full replace. Called after every points update.

`watchProfile` — live stream. Used by the Progression screen.

`getAllLevels` — returns all eight level records. Used by the Progression screen to render the full level map.

`getCurrentLevel` — returns the single record with `status: current`. Used by `ProgressionService` for quick level checks.

`saveLevel` — upsert. Called when `ProgressionService` updates `pointsEarned` or transitions a level's status.

`watchAllLevels` — live stream of all eight level records. Used by the Progression screen for live updates.

---

## Firestore Paths

```
/users/{userId}/progression/profile          ← single document
/users/{userId}/progressionLevels/{level}    ← 8 documents, level index as ID
```

Both covered by the existing security rule.

---

## Rules

- Called only by `ProgressionService`
- No business logic — only fetch and store
- `saveProfile` is always a full replace
- `watchProfile` and `watchAllLevels` detached when the screen is no longer visible