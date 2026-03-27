**File Name**: model_weekly_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

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
  updatedAt: DateTime
```

```
enum CupLevel { bronze, silver, gold, diamond }
```

- **id** — client-generated UUID. The Firestore document ID. Immutable.
- **weekStart** — identifies which week this cup belongs to. Used for queries, display ("week of March 3rd"), and `ScoringService` bonus trigger evaluation. Combined with user scope, guarantees one cup per week maximum.
- **cupLevel** — the tier earned. Determined by comparing `weeklyScore` against `AppConfig` thresholds.
- **weeklyScore** — the raw percentage score stored for auditability. Used by `ScoringService` for bonus trigger evaluation (e.g. diamond cup after a cupless week).
- **createdAt** — immutable.
- **updatedAt** — initialized to `createdAt`. Cups are append-only and never edited — `updatedAt` always equals `createdAt`. Both fields are present to satisfy the model convention.

---

## Cup Thresholds

Thresholds live in `AppConfig` as percentage values — consistent with the percentage scores returned by `PerformanceService.getOverallWeekScore()`. Default values:

|Cup|Minimum weekly score|
|---|---|
|🥉 Bronze|60%|
|🥈 Silver|75%|
|🥇 Gold|85%|
|💎 Diamond|95%|

Below 60% — no cup, no record.

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
- `updatedAt` initialized to `createdAt` — cups are never modified after creation
- Written only by `CupService`
- `toAchievementRecord()` called only by `AchievementService`