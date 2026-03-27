**File Name**: service_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** evaluates weekly performance and awards cups. An independent feature — not internal to Achievements. Publishes `CupEarnedEvent` carrying a `WeeklyCup` which implements `Achievable`. `AchievementService` subscribes and calls `toAchievementRecord()`.

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
publish CupEarnedEvent(cup: cup)
```

### `backfillMissingCups()` — on app startup

Runs once on startup to cover weeks where the device was inactive when `WeekEndedEvent` fired. Looks back `AppConfig.cupBackfillWeeks` (default: 8) weeks — consistent with the bonus trigger evaluation window in `ScoringService`. For each past week with no cup record, fetches score and runs the same logic silently. No `CupEarnedEvent` published for backfilled cups — they are historical records only.

The 8-week lookback is bounded, satisfying the windowed fetch rule. It is long enough to catch any missed weeks from a device that was offline for a month.

---

## Events Published

```
CupEarnedEvent
  cup: WeeklyCup    // implements Achievable
```

Consumed by `AchievementService` which calls `cup.toAchievementRecord()`.

---

## Pure Functions

### `_determineCupLevel(score)` → CupLevel?

Score is a percentage value (0–100) matching the return value of `PerformanceService.getOverallWeekScore()`.

```
if score >= AppConfig.cupThresholdDiamond: return diamond   // default: 95
if score >= AppConfig.cupThresholdGold:    return gold      // default: 85
if score >= AppConfig.cupThresholdSilver:  return silver    // default: 75
if score >= AppConfig.cupThresholdBronze:  return bronze    // default: 60
return null
```

Thresholds are percentage values in `AppConfig` — consistent with the percentage scores returned by `PerformanceService`.

---

## Read Functions

### `getAllCups()` → List<WeeklyCup>

Ordered by weekStart descending.

### `getCupsSince(from)` → List<WeeklyCup>

Returns cups earned since `from`. Called by `AchievementService.getCupsSince()` — `AchievementService` wraps this to provide a unified read interface across all achievement types. External callers use `AchievementService`, not `CupService` directly.

---

## Rules

- Independent feature — not internal to Achievements
- Cups based on raw `getOverallWeekScore()` — never accelerator-modified
- Idempotent — one cup per week maximum
- `WeeklyCup` implements `Achievable` — translation to `AchievementRecord` is the model's responsibility
- Backfill limited to `AppConfig.cupBackfillWeeks` — never fetches all history

---

## Dependencies

- `TemporalHelperService` — subscribes to `onWeekEnded`
- `PerformanceService.getOverallWeekScore()` — raw weekly score
- `CupRepository` — reads and writes `WeeklyCup`
- `AppConfig` — cup threshold constants, `cupBackfillWeeks`