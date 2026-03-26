**File Name**: interface_achievable **Feature**: Core **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** defines the contract any feature must fulfill to participate in the achievement system. A feature that implements `Achievable` on its domain model is saying: "this model represents something worth recording as an achievement." `AchievementService` calls `toAchievementRecord()` without knowing anything about the feature's internals.

---

## The Interface

```dart
abstract class Achievable {
  AchievementRecord toAchievementRecord();
}
```

One method. No other requirements. The feature decides everything else — its own models, repositories, services, events. The only obligation is that when `AchievementService` receives an event carrying an `Achievable`, it can call this method and get a fully populated `AchievementRecord`.

---

## What Implements It

Any domain model that represents an earned achievement implements `Achievable`:

|Model|Feature|Achievement type|
|---|---|---|
|`WeeklyCup`|Cups|cup|
|`MilestoneRecord`|Milestones|streakMilestone|
|`RewardRecord`|Rewards|reward|

`StreakRecord` does **not** implement `Achievable` — a streak record is continuous state, not a discrete achievement moment. `MilestoneRecord` (owned by the independent Milestones feature) is the achievement — it is created when a streak crosses a threshold.

---

## What Lives in Infrastructure

Because lower-level features implement `Achievable` and return `AchievementRecord`, both `AchievementRecord` and `AchievementType` must live in Infrastructure — below all features that reference them. This avoids any upward dependency.

```
Infrastructure owns:
  Achievable interface
  AchievementRecord model
  AchievementType enum
```

---

## Design Rationale

**Why an interface rather than a shared event shape?** An event shape enforces nothing at compile time — any feature can publish any payload. `Achievable` makes the contract explicit: the compiler rejects a model that doesn't implement `toAchievementRecord()`. Adding a new achievement-producing feature means implementing one method — the rest of the system requires no changes.

**Why one method?** `AchievementService` needs exactly one thing from each feature: a populated `AchievementRecord`. Everything else is the feature's internal concern. A larger interface would couple `AchievementService` to feature internals — the opposite of what we want.

**Why on the model, not the service?** The achievement moment is represented by a specific model instance — a `WeeklyCup` that was just earned, a `MilestoneRecord` that was just written. The model has all the data needed. The service doesn't — it would need to reconstruct context. Putting `toAchievementRecord()` on the model keeps the data and the translation together.

---

## Rules

- `Achievable` and `AchievementRecord` live in Infrastructure — never in a feature
- Only domain models implement `Achievable` — never services or repositories
- `toAchievementRecord()` is a pure function — no side effects, no async, no service calls
- `AchievementService` is the only caller of `toAchievementRecord()`