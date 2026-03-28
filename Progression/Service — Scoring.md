**File Name**: service_scoring **Feature**: Progression (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the translation layer between achievements and points. Subscribes to `AchievementEarnedEvent`, maps each `AchievementSubtype` to a point value via `AppConfig`, and publishes `PointsAwardedEvent` for `ProgressionService` to consume. Internal to Progression — never called by features outside.

---

## Design Decisions

**Why ScoringService is separate from ProgressionService.** `ProgressionService` only knows about points and levels. `ScoringService` owns the knowledge of what each achievement is worth. Changing point values only touches `AppConfig`. Adding a new achievement subtype only touches the lookup table here and one `AppConfig` entry. `ProgressionService` is untouched in both cases.

**Why the lookup is `subtype → points`, not `type → points`.** Different achievements of the same type have different values — a diamond cup is worth more than a bronze cup. Subtype is the specific semantic label, making it the right key for point lookup.

**Why the bonus system was removed.** The original design had a separate bonus trigger system tracking last bonus month. Every achievement now has one fixed point value in `AppConfig`. The scoring system is one clean table.

---

## Events Subscribed

### `AchievementService.onAchievementEarned` → `_onAchievementEarned(event)`

```
points = _lookupPoints(event.record.subtype)
if points == 0: return   // unrecognised subtype — logged as warning

publish PointsAwardedEvent(
  points: points,
  subtype: event.record.subtype,
  createdAt: event.record.createdAt,
)
```

---

## Events Published (internal to Progression)

```
PointsAwardedEvent
  points: double
  subtype: AchievementSubtype
  createdAt: DateTime
```

Consumed only by `ProgressionService`. Never published externally.

---

## Lookup Table

### `_lookupPoints(subtype)` → double

```
// cup achievements
bronzeCup        → AppConfig.pointsBronzeCup
silverCup        → AppConfig.pointsSilverCup
goldCup          → AppConfig.pointsGoldCup
diamondCup       → AppConfig.pointsDiamondCup

// streak achievements
globalBestStreak → AppConfig.pointsGlobalBestStreak
threeDay         → AppConfig.pointsThreeDay
fiveDay          → AppConfig.pointsFiveDay
sevenDay         → AppConfig.pointsSevenDay
tenDay           → AppConfig.pointsTenDay
fourteenDay      → AppConfig.pointsFourteenDay

// garment achievements
garmentCompleted → AppConfig.pointsGarmentCompleted

// reward achievements — placeholder for future Rewards feature
periodicReward   → AppConfig.pointsPeriodicReward
```

Returns 0.0 for unrecognised subtypes — never throws. Logs a warning when 0 is returned so gaps are visible during development.

**All point values are placeholders.** They will be carefully designed after launch based on real user earning rates, desired progression pace, and which achievements should feel most significant. Only `AppConfig` needs to change — this lookup table and `ProgressionService` are untouched.

---

## Rules

- Internal to Progression — never called by features outside
- All point values from `AppConfig` — never hardcoded
- Unrecognised subtypes return 0 points and log a warning — never throw
- No bonus system, no separate state — one clean subtype-to-points table
- No dependency on `ProgressionService` — internal event is sufficient

---

## Dependencies

- `AchievementService` — subscribes to `onAchievementEarned`
- `AppConfig` — all point values