**File Name**: model_weekly_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** a permanent record of one weekly cup earned. Append-only — never edited after creation.

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

- **id** — `year_weeknumber` format (e.g. `2026_W13`). Natural unique key per week per user. Used as the Firestore document ID — upsert by this ID is inherently idempotent. Immutable.
- **weekStart** — Monday 00:00:00 of the week this cup covers. Identifies which week.
- **cupLevel** — the tier earned. Determined by `weeklyScore` vs `AppConfig` thresholds.
- **weeklyScore** — raw percentage score stored for auditability and detail display.
- **createdAt** — immutable.
- **updatedAt** — initialized to `createdAt`. Cups are append-only — always equals `createdAt`.

---

## Cup Thresholds

Thresholds live in `AppConfig` as percentage values.

|Cup|Minimum weekly score|
|---|---|
|🥉 Bronze|60%|
|🥈 Silver|75%|
|🥇 Gold|85%|
|💎 Diamond|95%|

Below 60% — no cup, no record.

---

## Rules

- Append-only — never edited after creation
- `updatedAt` initialized to `createdAt`
- Written only by `CupService`