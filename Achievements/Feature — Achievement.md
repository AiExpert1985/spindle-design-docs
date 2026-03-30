**File Name**: feature_achievements **Phase**: 2 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

# Feature — Achievements

Achievements is the collection point for every meaningful moment the user earns across all features. It does not detect achievements — it receives them. Any feature that reaches an achievement moment calls `addAchievement()` downward. Achievements stores the record, publishes the event, and serves all achievement reads from a single collection.

---

## Why It Exists

Without a unified achievement store, each producing feature would need to answer its own history queries. Any screen showing mixed achievement history would have to call multiple features and assemble the results. Achievements inverts this — every achievement flows through one write path and is stored in one collection. Any consumer gets a consistent, filterable, time-windowed view of the full history in one call, regardless of which feature produced each record.

---

## Position in the System

Sits above the storage layer only — `AchievementRepository` is its sole dependency below. Features that produce achievements (cups, streak milestones, garment completions) sit above it and call down into it. Features that consume achievement data for scoring, display, or encouragement sit further above and read from it.

---

## How It Works

**One write path.** `addAchievement(AchievementRecord)` is the only entry point. It checks idempotency, writes the record, and publishes `AchievementEarnedEvent`. Every achievement — cup, streak milestone, global best streak, garment completion — enters the system through this single function.

**Why producing features call down, not the other way around.** The original design had Achievements sitting above Cups, Streak, and Garment, subscribing to their events:

```
Cups    → CupEarnedEvent          → AchievementService subscribes
Streak  → GlobalBestStreakEvent   → AchievementService subscribes
Garment → GarmentCompletedEvent   → AchievementService subscribes
```

This had a hidden contradiction. Achievements needed to sit above producers to subscribe to their events — but Progression needed to sit above Achievements to subscribe to `AchievementEarnedEvent`. This forced a tall chain where Achievements was both a subscriber looking down and a publisher looking up. Worse, if Achievements sat above Cups and Streak, any read for cup history or best streak had to proxy back down to those features — a middleman that added complexity without adding value.

The insight: an aggregator that stores everything it receives owns all the data it needs. If every achievement flows through `addAchievement()`, the `AchievementRecord` collection becomes the single source of truth. Cup history is achievements where `type == cup`. Best streak is achievements where `subtype == globalBestStreak`. No proxying, no external reads.

Moving Achievements below the producers resolved everything:

```
Cups    → AchievementService.addAchievement()
Streak  → AchievementService.addAchievement()
Garment → AchievementService.addAchievement()
```

Producing features call downward — valid in the dependency chain. Achievements reads from its own collection — no upward calls. Progression subscribes to `AchievementEarnedEvent` downward — valid. Adding a new producing feature requires zero changes here.

This is an inversion of the typical observer pattern. Instead of the aggregator watching producers (pull via events), producers notify the aggregator directly (push via direct call). The aggregator becomes a write-through store — one write door, all reads self-served from the stored collection. In formal terms this maps to the write side of CQRS: `addAchievement()` is the command path, `getAchievements()` is the query path, and the `AchievementRecord` collection is the single source of truth for both.

**Records are permanent.** Achievements are facts about the user's history. Deleting a commitment does not erase what was accomplished with it. `sourceId` may become stale if the originating record is deleted, but the achievement remains valid and displayable.

**One read interface.** `getAchievements(from, to, type?, subtype?, definitionId?)` covers every read case — cups history, streak records, garment completions — all answered from the same collection with the same function. No producing feature is called for reads.

---

## Achievement Types and Subtypes

Every `AchievementRecord` carries a `type` (which feature produced it) and a `subtype` (what specifically happened). Both are stored as strings derived from Dart enum `.name` — compile-time safe, no manual string literals anywhere.

**One enhanced enum per feature group, one file per group:**

```
achievement_cup.dart      → enum CupAchievement
achievement_streak.dart   → enum StreakAchievement
achievement_garment.dart  → enum GarmentAchievement
```

Each enum value carries its own `description` as a compile-time constant — no lookup table, no switch statement, no separate map. The description lives exactly where the achievement is defined.

Example of how one group is structured:

```dart
// achievement_cup.dart
enum CupAchievement {
  bronze('Kept 60% of your commitments this week'),
  silver('Kept 75% of your commitments this week'),
  gold('Kept 85% of your commitments this week'),
  diamond('Kept 95% of your commitments this week');

  const CupAchievement(this.description);
  final String description;
  String get type => 'cup';
}
```

All other groups follow the same pattern in their own files.

**Why enhanced enums, not sealed classes.** Dart enums are already sealed — they cannot be subclassed. Enhanced enums support fields, constructors, and methods, giving all the structure needed without the serialization complexity of sealed classes. Storing and retrieving is trivial: `.name` serializes, `.byName()` deserializes. Both are built-in Dart properties — compile-safe, no string literals.

**Dynamic descriptions.** The one case needing runtime data is `StreakAchievement.milestone` — its description should say "kept for N days." This is handled by a `metadata` field (`Map<String, dynamic>?`) on `AchievementRecord`. The enum provides the base description; the display layer reads `metadata['streakCount']` to augment it. One field handles all current and future dynamic cases.

**Adding a new achievement subtype** — add one enum value with its description to the right file. One `AppConfig` point entry. One display mapping. Nothing else changes.

---

## Events

**`AchievementEarnedEvent`** — published after every successful write. Carries the full `AchievementRecord`. Consumed by Progression for point scoring and by features above that respond to achievement moments.

---

## Rules

- `addAchievement()` is the only write path
- No event subscriptions — producing features call directly
- `AchievementEarnedEvent` published only after successful write
- All reads are self-contained — no calls to any other feature
- Records are permanent — never deleted for any reason
- All reads use a time window — never unbounded
- `type` and `subtype` stored as enum `.name` strings — no manual string literals
- Adding a new producing feature or achievement subtype requires zero changes to this feature

---

## Later Improvements

**Rewards feature.** A `RewardService` that evaluates sustained rolling performance and calls `addAchievement()` with `type: reward`. The `periodicReward` subtype and its `AppConfig` point entry are already present — fully additive when ready.