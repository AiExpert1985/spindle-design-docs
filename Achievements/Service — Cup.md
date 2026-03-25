**File Name**: service_cup **Feature**: Achievements (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** evaluates weekly performance and awards cups. Internal to Achievements — never called directly by features outside the feature boundary. Publishes an internal event consumed by `AchievementService`.

---

## How Cups Are Earned

At the end of each week, `CupService` reads the overall weekly performance score and compares it against `AppConfig` thresholds. If the score meets the lowest threshold, a cup is awarded at the appropriate level. One cup per week maximum. Below the minimum threshold — no cup, no record, no event.

Cups are based on raw `PerformanceService.getOverallWeekScore()` — never modified by the accelerator. The cup reflects honest weekly performance.

---

## Events Subscribed

### `WeekEndedEvent` → `_onWeekEnded(event)`

```
score = PerformanceService.getOverallWeekScore(event.weekStart)
level = _determineCupLevel(score)
if level == null: return   // below minimum threshold — no cup

if CupRepository.cupExistsForWeek(event.weekStart): return   // idempotent

cup = WeeklyCup(weekStart, cupLevel: level, weeklyScore: score)
CupRepository.saveCup(cup)
publish CupEarnedInternalEvent(cup)
```

### `backfillMissingCups()` — startup

Runs on app startup. For each past week with no cup record, fetches score and calls the same logic silently. No event published for backfilled cups.

---

## Events Published (internal)

```
CupEarnedInternalEvent
  cup: WeeklyCup
```

Consumed only by `AchievementService` within the Achievements feature.

---

## Pure Functions

### `_determineCupLevel(score)` → CupLevel?

```
if score >= AppConfig.cupThresholdDiamond: return diamond
if score >= AppConfig.cupThresholdGold:    return gold
if score >= AppConfig.cupThresholdSilver:  return silver
if score >= AppConfig.cupThresholdBronze:  return bronze
return null
```

---

## Read Functions

### `getAllCups()` → List<WeeklyCup>

Ordered by weekStart descending. Proxied through `AchievementService` for external callers.

### `getCupsSince(from)` → List<WeeklyCup>

Used by `ScoringService` for bonus trigger evaluation.

---

## Rules

- Internal to Achievements — never called directly by features outside
- Cup thresholds read from `AppConfig` — never hardcoded
- Idempotent — one cup per week
- No cup below `AppConfig.cupThresholdBronze`

---

## Dependencies

- EventBus — subscribes to `WeekEndedEvent`; publishes `CupEarnedInternalEvent`
- `PerformanceService.getOverallWeekScore()` — raw weekly score
- `CupRepository` — reads and writes `WeeklyCup`
- `AppConfig` — cup threshold constants