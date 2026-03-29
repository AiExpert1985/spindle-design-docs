**File Name**: model_weekly_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** a permanent record of one weekly cup earned. `CupService` constructs an `AchievementRecord` and calls `AchievementService.addAchievement()` when a cup is earned — this model has no knowledge of the achievement system.

---

## What a Weekly Cup Is

At the end of each week, `CupService` evaluates the overall weekly performance score. If the score meets a threshold, a cup is awarded at the appropriate level. The cup recognises consistent weekly performance — not any single commitment or day.

Cups are based on raw performance (`PerformanceService.getOverallWeekScore()`) — never on the accelerator-modified value.

---

## Fields

```dart
class WeeklyCup {
  final String id;
  final DateTime weekStart;
  final CupLevel cupLevel;
  final double weeklyScore;
  final DateTime createdAt;
  final DateTime updatedAt;
}

enum CupLevel { bronze, silver, gold, diamond }
```

- **id** — client-generated UUID. Immutable.
- **weekStart** — Monday 00:00:00 of the week this cup covers. Identifies which week.
- **cupLevel** — the tier earned. Determined by `weeklyScore` vs `AppConfig` thresholds.
- **weeklyScore** — raw percentage score stored for auditability and detail display.
- **createdAt** — immutable.
- **updatedAt** — initialized to `createdAt`. Cups are append-only — always equals `createdAt`.

---

## Cup Thresholds

Thresholds live in `AppConfig` as percentage values — consistent with `PerformanceService.getOverallWeekScore()` return values.

|Cup|Minimum weekly score|
|---|---|
|🥉 Bronze|60%|
|🥈 Silver|75%|
|🥇 Gold|85%|
|💎 Diamond|95%|

Below 60% — no cup, no record.

---

## Rules

- One record per week maximum — `CupService` checks idempotency before writing
- Append-only — never edited after creation
- `updatedAt` initialized to `createdAt`
- Written only by `CupService`