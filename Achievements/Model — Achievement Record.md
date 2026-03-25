**File Name**: model_achievement_record **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the unified display model for any achievement the user has earned. Written once at earn time — never updated. Allows the achievements screen and progression system to work with a single consistent format regardless of achievement type.

---

## What an Achievement Record Is

Every significant moment the user earns — a cup, a streak milestone, an occasional reward — is translated into one `AchievementRecord` by `AchievementService`. The record carries enough information to display the achievement without knowing anything about the source. If more detail is needed, the caller uses `sourceId` to fetch from the specific internal service.

This is a projection, not a live view. The source record (WeeklyCup, MilestoneRecord, RewardRecord) is the truth. The `AchievementRecord` is a snapshot written at earn time and never modified.

---

## Fields

```
AchievementRecord
  id: String
  type: AchievementType
  subtype: String            // specific variant within the type
  sourceId: String?          // id of the originating record — for detail fetches
  definitionId: String?      // null for cross-commitment achievements (cups, rewards)
  earnedAt: DateTime
  createdAt: DateTime
```

```
enum AchievementType {
  cup               // weekly performance cup
  streakMilestone   // streak count milestone reached
  reward            // occasional periodic reward
}
```

- **type** — the category of achievement. Drives point lookup in `AppConfig` and display in the achievements screen.
- **subtype** — the specific variant. Examples: `bronze`, `silver`, `gold`, `diamond` for cups. `three_day`, `five_day`, `seven_day`, `ten_day`, `fourteen_day` for streak milestones. `monthly` for rewards.
- **sourceId** — links back to the originating record. Null if the source has been permanently deleted (commitment deleted with its streak history). Used by the achievements screen to show detail on tap.
- **definitionId** — which commitment this achievement belongs to. Null for cups and rewards which are cross-commitment.
- **earnedAt** — when the achievement was earned. For cups: the Sunday night the week was evaluated. For milestones: the moment the streak threshold was crossed. For rewards: the moment the condition was met.

---

## Rules

- Append-only — never edited after creation
- Written only by `AchievementService` — never by internal services directly
- One record per achievement event — no duplicates for the same source event
- `sourceId` may become stale if the source is deleted — the record is still valid for display, `sourceId` is used for optional detail only
- Deleted when all data for this user is deleted — never deleted individually