**File Name**: model_global_best_streak_record **Feature**: Streak **Phase**: 2 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the single best streak ever achieved across all commitments. `StreakService` constructs an `AchievementRecord` and calls `AchievementService.addAchievement()` when a new global best crosses a milestone threshold — this model has no knowledge of the achievement system.

---

## Fields

```dart
class GlobalBestStreakRecord {
  final String definitionId;     // which commitment holds the all-time best
  final String commitmentName;   // snapshotted at write time
  final int streakDays;          // the best streak value
  final DateTime achievedAt;     // when this best was reached
  final DateTime createdAt;
  final DateTime updatedAt;
}
```

- **definitionId** — used as the document ID — no separate `id` field. Updated when a new best is set.
- **commitmentName** — snapshotted so the record stays readable after rename or deletion.
- **streakDays** — all-time best streak count across all commitments.
- **achievedAt** — when the best was reached. Used for display.
- **createdAt** — set on first ever personal best. Immutable.
- **updatedAt** — updated whenever a new best is set.

---

## Why Achievement Is Published Only at Milestone Values

`StreakService` calls `AchievementService.addAchievement()` for a new global best only when `streakDays` is a value in `AppConfig.streakMilestones`. Without this gate, a user on a 30-day streak would earn a new "personal best" achievement every single day from their previous best onward — diluting every one of them. At milestone values the achievement means something specific: the user has crossed a recognized threshold for the first time ever.

---

## Rules

- One record per user — `definitionId` is the document ID
- Updated whenever a new all-time best is set at a milestone value
- Written only by `StreakService`
- Never deleted — represents the user's lifetime achievement
- `commitmentName` snapshotted at write time — not updated on rename