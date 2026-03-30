**File Name**: model_achievement_record **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** the unified record for any achievement the user has earned. Written once — never updated. Producing features construct and pass this record to `AchievementService.addAchievement()`.

---

## Fields

```dart
class AchievementRecord {
  final String id;
  final String type;              // enum .name — e.g. 'cup', 'streak', 'garment'
  final String subtype;           // enum .name — e.g. 'bronze', 'streakMilestone'
  final String? sourceId;         // id of the originating record — for detail fetches
  final String? definitionId;     // null for cross-commitment achievements
  final Map<String, dynamic>? metadata;  // dynamic data for description augmentation
  final DateTime createdAt;
  final DateTime updatedAt;
}
```

- **type** — which feature produced this achievement. Stored as the enum `.name` string. Used for filtering.
- **subtype** — what specifically happened. Stored as the enum `.name` string. Used by the scoring service for point lookup and by the display layer for label and icon.
- **sourceId** — links to the originating record for detail fetches. May become stale if source is deleted — achievement remains valid.
- **definitionId** — which commitment this belongs to. Null for cross-commitment achievements (cups, global best streak).
- **metadata** — optional dynamic data for subtypes whose display description needs runtime values. Example: `{'streakCount': 9}` for `streakMilestone`. Null for subtypes with fixed descriptions.
- **createdAt** — when `AchievementService` wrote this record. Immutable. Used as achievement date for display.
- **updatedAt** — initialized to `createdAt`. Records are append-only — always equals `createdAt`.

---

## Achievement Enums

One enhanced enum per feature group, one file per group. Each value carries a compile-time `description` and a `type` getter. See `feature_achievements` for the full design and file structure.

```dart
// achievement_cup.dart
enum CupAchievement { bronze, silver, gold, diamond }

// achievement_streak.dart
enum StreakAchievement { streakMilestone, globalBestStreak }

// achievement_garment.dart
enum GarmentAchievement { garmentCompleted, garmentFortified, garmentGolded, garmentDiamond }
```

Serialization: `enumValue.name` → stored string. Deserialization: `Enum.values.byName(stored)` → enum value. Both are built-in Dart properties — no manual string literals.

---

## Why `type` and `subtype` Are Stored as Strings

`AchievementRecord` stores the enum `.name` strings rather than the enum values themselves. Firestore has no native enum support — strings are the correct persistence format. The enum files are the compile-time source of truth; the stored strings are their serialized form.

---

## Rules

- Owned by the Achievements feature — not a shared domain type
- Append-only — never edited after creation
- `updatedAt` initialized to `createdAt`
- Written only by `AchievementService.addAchievement()`
- Never deleted — achievement history is permanent
- `type` and `subtype` stored as enum `.name` — no manual string literals anywhere