**File Name**: model_progression_profile **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the user's current progression state. One record per user. Contains current points toward the active level and the cached current level.

---

## Design Rationale

**Why points reset per level (carry-over model).** When the user levels up, the points required for the completed level are subtracted. Any excess carries over into the next level. Example: level requires 10 points, user has 13 — they start the next level with 3 points already earned.

This means `currentPoints` always represents progress toward the current level, not a lifetime total. The user always sees a clear, answerable question: "I have X points, I need Y for the next level." A running lifetime total is less motivating — it grows forever and the gap to the next level is harder to feel.

**Why `bonusPointsTotal` was removed.** The original design had bonus points as a separate category. This added complexity without clarity — users don't think in "base points" vs "bonus points." Every achievement has one point value. The scoring system is one clean table. One points field is sufficient.

**Why `currentLevel` is cached.** Level is derived from the `LevelRecord` collection — whichever record has `status: current`. Caching it here avoids a collection query on every read. It is always kept in sync when `ProgressionService` updates the profile.

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