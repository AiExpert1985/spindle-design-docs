**File Name**: service_scoring **Feature**: Progression (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the translation layer between achievements and points. Subscribes to `AchievementEarnedEvent`, converts each achievement to a point value using `AppConfig`, and publishes `PointsAwardedEvent` for `ProgressionService` to consume. Internal to Progression — `ProgressionService` never reads from Achievements directly.

---

## Design Decisions

**Why ScoringService is separate from ProgressionService.** `ProgressionService` only knows about points and levels. `ScoringService` owns the knowledge of what each achievement is worth. Adding a new achievement type or changing point values only touches `ScoringService` and `AppConfig` — `ProgressionService` is untouched.

**Why `lastBonusAwardedMonth` lives in `ScoringState`, not `ProgressionProfile`.** It is operational state for the scoring logic, not a description of the user's journey. `ProgressionProfile` describes what the user has achieved. `ScoringState` describes what the scoring mechanics last did. Keeping them separate means `ProgressionProfile` remains a clean domain model.

---

## Events Subscribed

### `AchievementEarnedEvent` → `_onAchievementEarned(event)`

```
points = _lookupPoints(event.record.type, event.record.subtype)
if points == 0: return   // unrecognised subtype — no points awarded

publish PointsAwardedEvent(
  points: points,
  achievementType: event.record.type,
  achievementSubtype: event.record.subtype,
  earnedAt: event.record.earnedAt,
  isBonus: false
)

if event.record.type == cup:
  _evaluateAndPublishBonus(event.record)
```

---

## Events Published (internal)

```
PointsAwardedEvent
  points: double
  achievementType: AchievementType
  achievementSubtype: String
  earnedAt: DateTime
  isBonus: bool
```

`isBonus: true` tells `ProgressionService` to also update `bonusPointsTotal`.

---

## Pure Functions

### `_lookupPoints(type, subtype)` → double

```
cup + bronze    → AppConfig.cupPointsBronze
cup + silver    → AppConfig.cupPointsSilver
cup + gold      → AppConfig.cupPointsGold
cup + diamond   → AppConfig.cupPointsDiamond
streakMilestone → AppConfig.milestonePoints
reward          → AppConfig.rewardPoints
```

Returns 0.0 for unrecognised subtypes — never throws.

---

## Bonus Trigger Evaluation

### `_evaluateAndPublishBonus(cupRecord)` → void

Called only for cup achievements.

```
state = ScoringRepository.getState()
if state != null and isSameMonth(state.lastBonusAwardedMonth, now):
  return   // already awarded a bonus this month

cupHistory = AchievementService.getCupsSince(thirtyDaysAgo)
bonus = _checkBonusTriggers(cupRecord, cupHistory)
if bonus == 0: return

// update ScoringState
state = state ?? ScoringState()
state.lastBonusAwardedMonth = firstDayOfCurrentMonth()
ScoringRepository.saveState(state)

publish PointsAwardedEvent(
  points: bonus,
  achievementType: cup,
  achievementSubtype: _bonusTriggerName(cupRecord, cupHistory),
  earnedAt: now,
  isBonus: true
)
```

### `_checkBonusTriggers(cupRecord, cupHistory)` → double

Evaluates triggers highest bonus first — only first match fires.

|Priority|Trigger|Bonus|
|---|---|---|
|1|Diamond cup after a cupless week|`AppConfig.bonusPointsDiamondComeback`|
|2|First diamond cup ever|`AppConfig.bonusPointsFirstDiamond`|
|3|First cup ever earned|`AppConfig.bonusPointsFirstCup`|
|4|Cup after 2+ cupless weeks (comeback)|`AppConfig.bonusPointsComeback`|
|5|3 cups in 3 consecutive weeks|`AppConfig.bonusPointsThreeConsecutive`|

Returns 0.0 if no trigger matches.

---

## Rules

- Internal to Progression — never called by features outside
- All point values from `AppConfig` — never hardcoded
- Unrecognised achievement subtypes award 0 points — never throws
- Bonus evaluation only for cup achievements
- One bonus per calendar month — enforced via `ScoringState.lastBonusAwardedMonth`
- No dependency on `ProgressionService` — scoring state is fully self-contained

---

## Dependencies

- EventBus — subscribes to `AchievementEarnedEvent`; publishes `PointsAwardedEvent`
- `ScoringRepository` — reads and writes `ScoringState`
- `AchievementService.getCupsSince()` — bonus trigger evaluation
- `AppConfig` — all point values and bonus trigger definitions