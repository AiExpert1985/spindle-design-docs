**File Name**: model_weekly_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** a permanent record of one weekly cup earned. Implements `Achievable` — `AchievementService` calls `toAchievementRecord()` when a `CupEarnedEvent` arrives.

---

## What a Weekly Cup Is

At the end of each week, `CupService` evaluates the overall weekly performance score. If the score meets a threshold, a cup is awarded at the appropriate level. The cup is a recognition of consistent weekly performance — not of any single commitment or day.

Cups are based on raw performance (`PerformanceService.getOverallWeekScore()`) — never on the accelerator-modified value. The cup reflects what the user actually did, not how fast their garment is growing.

---

## Fields

```
WeeklyCup
  id: String
  weekStart: DateTime     // Monday 00:00:00 of the week this cup covers
  cupLevel: CupLevel
  weeklyScore: double     // the raw overall week score that earned this cup
  createdAt: DateTime
```

```
enum CupLevel { bronze, silver, gold, diamond }
```

- **weekStart** — the unique identifier for this week. Combined with user scope, guarantees one cup per week maximum.
- **cupLevel** — the tier earned. Determined by comparing `weeklyScore` against `AppConfig` thresholds.
- **weeklyScore** — the raw score stored for auditability and display. Used by `ScoringService` for bonus trigger evaluation (e.g. diamond cup after a cupless week).

---

## Cup Thresholds

Thresholds live in `AppConfig` — never hardcoded. Default values:

|Cup|Minimum weekly score|
|---|---|
|🥉 Bronze|0.60|
|🥈 Silver|0.75|
|🥇 Gold|0.85|
|💎 Diamond|0.95|

Below 0.60 — no cup, no record.

---

## Achievable Implementation

```dart
@override
AchievementRecord toAchievementRecord() {
  return AchievementRecord(
    type: AchievementType.cup,
    subtype: cupLevel.name,       // 'bronze' | 'silver' | 'gold' | 'diamond'
    sourceId: id,
    definitionId: null,           // cups are cross-commitment
    earnedAt: createdAt,
  );
}
```

Pure function — no side effects, no service calls.

---

## Rules

- One record per week maximum — `CupService` checks idempotency before writing
- Append-only — never edited after creation
- Written only by `CupService`
- `toAchievementRecord()` called only by `AchievementService`