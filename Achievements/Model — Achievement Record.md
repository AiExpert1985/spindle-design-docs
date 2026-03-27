**File Name**: model_achievement_record **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the unified record for any achievement the user has earned. Owned by the Achievements feature. Producing features construct and pass this record to `AchievementService.addAchievement()`. Written once — never updated.

---

## Design Rationale

**Why `AchievementRecord` lives in the Achievements feature, not the Domain layer.** Achievements sits below all producing features in the dependency chain. Producing features call `AchievementService.addAchievement()` downward and reference `AchievementRecord` as a downward dependency. No upward coupling needed.

**Why no `Achievable` interface.** The achievement moment is a service-level decision, not a model-level one. Each producing service constructs the `AchievementRecord` directly via a private `_addAchievement()` function at the exact moment the condition is met. The contract is enforced by the function signature, not an interface. This is simpler, more explicit, and keeps each service fully in control of its own achievement logic.

**Why `type` and `subtype` are both enums.** `type` identifies the source feature. `subtype` identifies what specifically happened. `ScoringService` maps each `subtype` to a point value in `AppConfig`. Enums are compile-time safe — a missing subtype is a compile error, not a silent 0-point award. Adding a new achievement means one new enum value and one `AppConfig` entry.

**Why `AchievementService` is the single source of truth.** Every achievement flows through `addAchievement()`. All queries — cups history, streak records, best streak, garment completions — are answered from the `AchievementRecord` collection. No producing feature needs to be called for achievement data.

---

## Fields

```dart
class AchievementRecord {
  final String id;
  final AchievementType type;
  final AchievementSubtype subtype;
  final String? sourceId;       // id of the originating record — for detail fetches
  final String? definitionId;   // null for cross-commitment achievements
  final DateTime createdAt;
  final DateTime updatedAt;
}
```

- **type** — source feature. Used for filtering in the achievements screen.
- **subtype** — specific achievement. Used by `ScoringService` for point lookup and by the screen for display label and icon.
- **sourceId** — links to the originating record for detail fetches. May become stale if source is deleted — record remains valid.
- **definitionId** — which commitment this belongs to. Null for cross-commitment achievements.
- **createdAt** — when `AchievementService` wrote this record. Immutable. Used as achievement date for display.
- **updatedAt** — initialized to `createdAt`. Records are append-only — always equals `createdAt`.

---

## Enums

```dart
/// Source feature — which feature produced this achievement.
enum AchievementType {
  cup,
  streak,
  garment,
  reward,     // future — see Later Improvements
}

/// Specific achievement — semantic label for what happened.
/// Each value maps to a point value in AppConfig.
///
/// IMPORTANT: current values are placeholders. The full set and point values
/// will be finalized after launch based on real user data. Adding a new subtype:
/// 1. One new enum value here
/// 2. One AppConfig point entry
/// 3. One display mapping in the achievements screen
/// Nothing else changes.
enum AchievementSubtype {
  // cup achievements — weekly performance thresholds
  bronzeCup,
  silverCup,
  goldCup,
  diamondCup,

  // streak achievements — personal bests and milestone thresholds
  globalBestStreak,   // new all-time best at a milestone threshold
  threeDay,           // 3 consecutive kept windows
  fiveDay,
  sevenDay,
  tenDay,
  fourteenDay,

  // garment achievements — visual completion events
  garmentCompleted,   // garment reached 100% Do or 0% Avoid

  // reward achievements — future feature, placeholder kept intentionally
  // so ScoringService and the achievements screen are ready when Rewards ships
  periodicReward,
}
```

---

## Why `milestone` Was Removed from AchievementType

Milestones were originally a separate feature that watched `StreakChangedEvent` and translated streak thresholds into achievements. This was a legacy design from before the current achievement system existed. Now `StreakService` detects milestone thresholds directly via its internal `_addAchievement()` function and records them with `type: streak`. The Milestone feature and its event are gone. Streak milestone achievements now correctly belong under the `streak` type — they are streak-derived events, not a separate concern.

---

## Rules

- Owned by the Achievements feature — not a shared domain type
- Append-only — never edited after creation
- `updatedAt` initialized to `createdAt`
- Written only by `AchievementService.addAchievement()`
- Never deleted — achievement history is permanent

---

## Later Improvements

**Rewards feature.** A `RewardService` that evaluates sustained rolling performance (e.g. 30 days above threshold) and calls `AchievementService.addAchievement()` with `type: reward, subtype: periodicReward`. Adding this feature requires zero changes to `AchievementService`, `ScoringService`, or the achievements screen — the enum value and `AppConfig` point entry are already present. The feature is fully additive.