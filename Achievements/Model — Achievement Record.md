**File Name**: model_achievement_record **Layer**: Domain **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the unified display model for any achievement the user has earned. Lives in the Domain layer so that any feature implementing `Achievable` can reference it without upward coupling. Written once at earn time by `AchievementService` — never updated.

---

## Fields

```dart
class AchievementRecord {
  final String id;
  final AchievementType type;
  final String subtype;       // specific variant — e.g. 'gold', '7_day', 'periodic'
  final String? sourceId;     // id of the originating model — for detail fetches
  final String? definitionId; // null for cross-commitment achievements
  final DateTime createdAt;
  final DateTime updatedAt;
}

enum AchievementType {
  cup,
  streakMilestone,
  reward,
}
```

- **type** — the category. Drives point lookup in `AppConfig` and display grouping.
- **subtype** — the specific variant within the type. Format: `bronze`, `silver`, `gold`, `diamond` for cups. `${streakCount}_day` (e.g. `3_day`, `7_day`, `14_day`) for milestones. `periodic` for rewards.
- **sourceId** — links back to the originating model. Used by the achievements screen to fetch detail on tap. May become stale if the source is deleted — record remains valid for display.
- **definitionId** — which commitment this belongs to. Null for cups and rewards which are cross-commitment.
- **createdAt** — when `AchievementService` wrote this record. Immutable.
- **updatedAt** — initialized to `createdAt`. Achievement records are append-only and never edited — `updatedAt` always equals `createdAt`. Both fields are present to satisfy the model convention.

---

## Rules

- Lives in the Domain layer — referenced by any feature implementing `Achievable`
- Append-only — never edited after creation
- `updatedAt` initialized to `createdAt` — records are never modified after creation
- Written only by `AchievementService` via `Achievable.toAchievementRecord()`
- `sourceId` may become stale on commitment deletion — record remains valid for display
- Never deleted based on commitment deletion — achievement history is permanent