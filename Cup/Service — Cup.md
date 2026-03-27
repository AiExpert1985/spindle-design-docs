**File Name**: service_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** evaluates weekly performance and awards cups. An independent feature. When a cup is earned, constructs an `AchievementRecord` and calls `AchievementService.addAchievement()` directly — no event publishing needed for the achievement handoff.

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

AchievementService.addAchievement(AchievementRecord(
  type: AchievementType.cup,
  subtype: _cupSubtype(level),   // bronzeCup | silverCup | goldCup | diamondCup
  sourceId: cup.id,
  definitionId: null,
  createdAt: now,
  updatedAt: now,
))
```

### `backfillMissingCups()` — on app startup

Looks back `AppConfig.cupBackfillWeeks` (default: 8) weeks. For each past week with no cup record, fetches score and runs the same logic silently. No achievement recorded for backfilled cups — they are historical records only. Backfill is bounded to prevent unbounded reads.

---

## Pure Functions

### `_determineCupLevel(score)` → CupLevel?

Score is a percentage value (0–100).

```
if score >= AppConfig.cupThresholdDiamond: return diamond   // default: 95
if score >= AppConfig.cupThresholdGold:    return gold      // default: 85
if score >= AppConfig.cupThresholdSilver:  return silver    // default: 75
if score >= AppConfig.cupThresholdBronze:  return bronze    // default: 60
return null
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

Ordered by weekStart descending. Used when cup detail is needed beyond what `AchievementService` provides.

### `getCupsSince(from)` → List<WeeklyCup>

Returns cups earned since `from`.

### `cupExistsForWeek(weekStart)` → bool

Internal idempotency check.

---

## Rules

- Cups based on raw `getOverallWeekScore()` — never accelerator-modified
- Idempotent — one cup per week maximum
- No achievement recorded for backfilled cups
- Thresholds are percentage values in `AppConfig`

---

## Dependencies

- `TemporalHelperService` — subscribes to `onWeekEnded`
- `PerformanceService.getOverallWeekScore()` — raw weekly score
- `AchievementService.addAchievement()` — records the achievement
- `CupRepository` — reads and writes `WeeklyCup`
- `AppConfig` — cup threshold constants, `cupBackfillWeeks`