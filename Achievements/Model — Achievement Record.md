**File Name**: model_achievement_record **Feature**: Infrastructure **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the unified display model for any achievement the user has earned. Lives in Infrastructure so that any feature implementing `Achievable` can reference it without upward coupling. Written once at earn time by `AchievementService` — never updated.

---

## Fields

```dart
class AchievementRecord {
  final String id;
  final AchievementType type;
  final String subtype;       // specific variant — e.g. 'gold', 'seven_day', 'periodic'
  final String? sourceId;     // id of the originating model — for detail fetches
  final String? definitionId; // null for cross-commitment achievements
  final DateTime earnedAt;
  final DateTime createdAt;
}

enum AchievementType {
  cup,
  streakMilestone,
  reward,
}
```

- **type** — the category. Drives point lookup in `AppConfig` and display grouping.
- **subtype** — the specific variant within the type. Examples: `bronze`, `silver`, `gold`, `diamond` for cups. `three_day`, `five_day`, `seven_day`, `ten_day`, `fourteen_day` for milestones. `periodic` for rewards.
- **sourceId** — links back to the originating model. Used by the achievements screen to fetch detail on tap. May become stale if the source is deleted — still valid for display.
- **definitionId** — which commitment this belongs to. Null for cups and rewards which are cross-commitment.
- **earnedAt** — when the achievement happened. Set by the implementing model in `toAchievementRecord()`.

---

## Rules

- Lives in Infrastructure — referenced by any feature implementing `Achievable`
- Append-only — never edited after creation
- Written only by `AchievementService` via `Achievable.toAchievementRecord()`
- `sourceId` may become stale on commitment deletion — record remains valid for display