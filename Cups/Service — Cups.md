**File Name**: service_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 30-Mar-2026

---

**Purpose:** evaluates weekly performance and awards cups. Records the achievement via `AchievementService.addAchievement()` on every cup earned.

---

## Events Subscribed

### `WeekEndedEvent` → `_onWeekEnded(event)`

```
score = PerformanceService.getOverallWeekScore(event.weekStart)
level = _determineCupLevel(score)
if level == null: return   // below minimum threshold

cup = WeeklyCup(
  id: _weekId(event.weekStart),   // year_weeknumber — natural unique key
  weekStart: event.weekStart,
  cupLevel: level,
  weeklyScore: score,
)
CupRepository.saveCup(cup)   // upsert by id — idempotent by design
_addAchievement(cup)
```

---

## Internal Functions

### `_weekId(weekStart)` → String

```
return "${weekStart.year}_W${weekNumber(weekStart)}"
// e.g. "2026_W13"
```

`weekNumber` is derived from the user's configured `weekStartDay` via `TemporalHelperService`, not from locale-dependent ISO week numbering. This prevents misalignment for users whose week starts on Saturday or Sunday, and avoids edge cases at year boundaries where ISO and user-relative week numbers may differ.

The document ID is the natural unique key per week per user. Upsert by this ID means no existence check needed — writing the same week twice produces the same record.

### `_determineCupLevel(score)` → CupLevel?

Score is a percentage value (0–100).

```
if score >= AppConfig.cupThresholdDiamond: return diamond   // default: 95
if score >= AppConfig.cupThresholdGold:    return gold      // default: 85
if score >= AppConfig.cupThresholdSilver:  return silver    // default: 75
if score >= AppConfig.cupThresholdBronze:  return bronze    // default: 60
return null
```

### `_addAchievement(cup)`

```
AchievementService.addAchievement(AchievementRecord(
  id: "${cup.id}_${cup.cupLevel.name}",   // e.g. "2026_W13_bronze"
  type: CupAchievement.values.byName(cup.cupLevel.name).type,
  subtype: cup.cupLevel.name,
  definitionId: null,
  createdAt: now,
  updatedAt: now,
))
```

---

## Read Functions

### `getAllCups(from, to, definitionId?)` → List<WeeklyCup>

Returns cups within the date range. `definitionId` is always null for cups — parameter kept for interface consistency with the rest of the system. Ordered by `weekStart` descending.

---

## Rules

- Idempotent by document ID — `year_weeknumber` as the unique key, upsert replaces any duplicate
- Thresholds are percentage values in `AppConfig`

---

## Dependencies

- `TemporalHelperService` — subscribes to `WeekEndedEvent`
- `PerformanceService.getOverallWeekScore()` — raw weekly score
- `AchievementService.addAchievement()` — records the achievement
- `CupRepository` — reads and writes `WeeklyCup`
- `AppConfig` — cup threshold constants