**File Name**: service_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** evaluates weekly performance and awards cups. An independent feature — not internal to Achievements. Publishes `CupEarnedEvent` carrying a `WeeklyCup` which implements `Achievable`. `AchievementService` subscribes and calls `toAchievementRecord()`.

---

## Events Subscribed

### `WeekEndedEvent` → `_onWeekEnded(event)`

```
score = PerformanceService.getOverallWeekScore(event.weekStart)
level = _determineCupLevel(score)
if level == null: return   // below minimum threshold

if CupRepository.cupExistsForWeek(event.weekStart): return   // idempotent

cup = WeeklyCup(weekStart, cupLevel: level, weeklyScore: score)
CupRepository.saveCup(cup)
publish CupEarnedEvent(cup: cup)
```

### `backfillMissingCups()` — startup

For each past week with no cup record, fetches score and runs the same logic silently. No event published for backfilled cups.

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

Used by `ScoringService` via `AchievementService.getCupsSince()`.

---

## Rules

- Independent feature — not internal to Achievements
- Cups based on raw `getOverallWeekScore()` — never accelerator-modified
- Idempotent — one cup per week maximum
- `WeeklyCup` implements `Achievable` — translation to `AchievementRecord` is the model's responsibility

---

## Dependencies

- EventBus — subscribes to `WeekEndedEvent`; publishes `CupEarnedEvent`
- `PerformanceService.getOverallWeekScore()` — raw weekly score
- `CupRepository` — reads and writes `WeeklyCup`
- `AppConfig` — cup threshold constants