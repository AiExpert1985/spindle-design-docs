**File Name**: model_level_record **Feature**: Progression **Phase**: 3 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** one record per level per user. Together these eight records form the user's complete level map — showing which levels are done, which is active, and which are still ahead.

---

## Design Rationale

**Why a collection instead of a field on ProgressionProfile.** `ProgressionProfile` previously stored `levelAchievedDates` as a list. A separate collection is better because each level is a first-class object with its own status, points, and date. The screen can query all eight records and render the full map without any list index gymnastics. Adding new level metadata (flavor text, icon, date) requires no model changes — it is already a document.

**Why all eight records are created upfront.** Creating all levels at account initialization means the Progression screen always has a complete map to render — no conditional logic for "has this level been reached yet." Pending levels simply have `status: pending` and `achievedAt: null`. This is also consistent with the UX goal of showing the user the full path from day one.

**Why points carry over, not reset to zero.** See `model_progression_profile` for the full rationale. `pointsEarned` stores how many points the user accumulated toward this specific level — for `done` levels this equals `pointsRequired` (they completed it). For the `current` level this is the live progress. For `pending` levels it is 0.

---

## Fields

```dart
class LevelRecord {
  final String id;
  final int level;                  // 0–7
  final String name;                // Apprentice, Weaver, Journeyman, Artisan,
                                    // Craftsman, Master Weaver, Loom Keeper, Penelope
  final int pointsRequired;         // points needed to complete this level
  final double pointsEarned;        // points accumulated toward this level
  final LevelStatus status;         // done | current | pending
  final DateTime? achievedAt;       // when this level was completed — null if pending
  final DateTime createdAt;
  final DateTime updatedAt;
}

enum LevelStatus { done, current, pending }
```

- **level** — index 0–7. Immutable.
- **name** — the level's display name. Immutable.
- **pointsRequired** — from `AppConfig.levelThresholds`. Immutable.
- **pointsEarned** — live progress for `current`, final value for `done`, 0 for `pending`.
- **status** — exactly one record has `status: current` at all times. All previous levels are `done`. All future levels are `pending`.
- **achievedAt** — written once when the level transitions from `current` to `done`. Never overwritten.

---

## Level Names and Thresholds

Point thresholds live in `AppConfig.levelThresholds`. Names are fixed. Both will be tuned after launch.

|Level|Name|Points required (placeholder)|
|---|---|---|
|0|Apprentice|10|
|1|Weaver|20|
|2|Journeyman|30|
|3|Artisan|50|
|4|Craftsman|75|
|5|Master Weaver|100|
|6|Loom Keeper|150|
|7|Penelope|200|

**These numbers are placeholders.** They will be carefully designed after launch based on real user earning rates and desired progression pace. The structure supports any values — only `AppConfig` needs to change.

---

## Rules

- Eight records created at account initialization — one per level
- Exactly one record has `status: current` at all times
- `achievedAt` written once on level completion — never overwritten
- `pointsEarned` updated by `ProgressionService` on every `PointsAwardedEvent`
- Written only by `ProgressionService`