**File Name**: service_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** evaluates weekly performance and awards cups. When a cup is earned, records the achievement directly via `AchievementService.addAchievement()`. An independent feature — sits above Achievements in the chain.

---

## Events Subscribed

### `TemporalHelperService.onWeekEnded` → `_onWeekEnded(event)`

```
score = PerformanceService.getOverallWeekScore(event.weekStart)
level = _determineCupLevel(score)
if level == null: return   // below minimum threshold

if CupRepository.cupExistsForWeek(event.weekStart): return   // idempotent

cup = WeeklyCup(weekStart: event.weekStart, cupLevel: level, weeklyScore: score)
CupRepository.saveCup(cup)
_addAchievement(cup)
```

### `backfillMissingCups()` — on app startup

Looks back `AppConfig.cupBackfillWeeks` (default: 8) weeks. For each past week with no cup record, fetches score and awards a cup silently. No achievement recorded for backfilled cups — they are historical records only. Backfill is bounded to satisfy the windowed fetch rule.

---

## Internal Functions

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

Private. Constructs and submits an `AchievementRecord`.

```
AchievementService.addAchievement(AchievementRecord(
  type: AchievementType.cup,
  subtype: _cupSubtype(cup.cupLevel),
  sourceId: cup.id,
  definitionId: null,   // cups are cross-commitment
  createdAt: now,
  updatedAt: now,
))
```

### `_cupSubtype(level)` → AchievementSubtype

```
bronze  → AchievementSubtype.bronzeCup
silver  → AchievementSubtype.silverCup
gold    → AchievementSubtype.goldCup
diamond → AchievementSubtype.diamondCup
```

---

## Read Functions

### `getAllCups(from, to)` → List<WeeklyCup>

Ordered by weekStart descending.

### `getCupsSince(from)` → List<WeeklyCup>

Used internally and by `AchievementService` for reads beyond what `AchievementRecord` carries.

---

## Rules

- Cups based on raw `getOverallWeekScore()` — never accelerator-modified
- Idempotent — one cup per week maximum, checked before write
- No achievement recorded for backfilled cups
- Thresholds are percentage values in `AppConfig`

---

## Dependencies

- `TemporalHelperService` — subscribes to `onWeekEnded`
- `PerformanceService.getOverallWeekScore()` — raw weekly score
- `AchievementService.addAchievement()` — records the achievement
- `CupRepository` — reads and writes `WeeklyCup`
- `AppConfig` — cup threshold constants, `cupBackfillWeeks`