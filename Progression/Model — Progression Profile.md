**File Name**: model_progression_profile **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** stores the user's current progression state. One record per user. Holds current points toward the active level and the cached current level index.

---

## Fields

```dart
class ProgressionProfile {
  final String id;
  final double currentPoints;   // points accumulated toward the current level
  final int currentLevel;       // 0–7, cached — matches the LevelRecord with status: current
  final DateTime createdAt;
  final DateTime updatedAt;
}
```

- **currentPoints** — progress toward the active level. Resets (carry-over) on level-up. Never decreases.
- **currentLevel** — cached value of the current level index. Updated by `ProgressionService` on every level-up.
- **createdAt** — set once on first `PointsAwardedEvent`. Immutable.
- **updatedAt** — updated on every write.

---

## Rules

- One record per user — created on first `PointsAwardedEvent`, never recreated
- `currentPoints` never decreases
- `currentLevel` always matches the `LevelRecord` with `status: current`
- Written only by `ProgressionService`