**File Name**: service_progression **Feature**: Progression **Phase**: 3 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains the user's progression state. Subscribes to `PointsAwardedEvent` from `ScoringService`, updates `currentPoints` and the `LevelRecord` collection, detects level-ups, and publishes `LevelReachedEvent`. The only writer of `ProgressionProfile` and `LevelRecord`.

---

## Design Decisions

**Why ProgressionService knows nothing about achievements.** `ProgressionService` only knows about points. It never reads `AchievementRecord`, never calls `AchievementService`, and never inspects what produced the points. `ScoringService` translates achievements to points — `ProgressionService` receives a number and updates state. This separation means the progression mechanics are completely independent of the achievement system. Changing what achievements are worth, or adding new ones, never touches this service.

**Why the LevelRecord collection is updated live, not computed on read.** The Progression screen shows all eight levels with live progress. Computing this from scratch on every read would require re-reading all achievement history and re-calculating all points. Maintaining the `LevelRecord` collection live means the screen is always a direct read — no calculation needed. This is worth the extra writes.

**Why points carry over on level-up.** Resetting to zero on level-up would erase points the user legitimately earned. If a level requires 10 points and the user earned 13, they start the next level with 3 — the extra 3 points represent real effort and should count. This also prevents a cliff-edge feeling where a burst of activity that crosses a level boundary feels wasted.

---

## Events Subscribed

### `ScoringService.onPointsAwarded` → `_onPointsAwarded(event)`

```
profile = getOrCreateProfile()
currentLevel = getOrCreateCurrentLevel()

// add points to current level
currentLevel.pointsEarned += event.points
profile.currentPoints += event.points

// check for level-up
while currentLevel.pointsEarned >= currentLevel.pointsRequired
  and currentLevel.level < maxLevel:

  excessPoints = currentLevel.pointsEarned - currentLevel.pointsRequired
  currentLevel.pointsEarned = currentLevel.pointsRequired
  currentLevel.status = done
  currentLevel.achievedAt = now
  saveLevel(currentLevel)

  nextLevel = getLevelRecord(currentLevel.level + 1)
  nextLevel.status = current
  nextLevel.pointsEarned = excessPoints   // carry over
  saveLevel(nextLevel)

  profile.currentLevel = nextLevel.level
  profile.currentPoints = excessPoints
  currentLevel = nextLevel

  publish LevelReachedEvent(
    newLevel: nextLevel.level,
    levelName: nextLevel.name,
    previousLevel: nextLevel.level - 1,
  )

saveProfile(profile)
saveLevel(currentLevel)   // save final current level state
```

The `while` loop handles the edge case where a single achievement awards enough points to skip multiple levels — unlikely but structurally correct.

---

## Events Published

```
LevelReachedEvent
  newLevel: int
  levelName: String
  previousLevel: int
```

Published when a level-up occurs. Consumed by `EncouragementService` for celebration, and by `NotificationSchedulingService` for the level-up notification.

---

## Initialization

On first use (first `PointsAwardedEvent`):

1. Create `ProgressionProfile` with `currentPoints: 0`, `currentLevel: 0`
2. Create all eight `LevelRecord` entries — level 0 as `status: current`, levels 1–7 as `status: pending`

This initialization happens lazily on the first points award — no separate setup step needed.

---

## Read Functions

### `getProgressionSummary()` → ProgressionSummary

One-time read. Assembles `ProgressionProfile` + all `LevelRecord` entries into a single summary object for the Progression screen.

```
ProgressionSummary
  currentPoints: double
  currentLevel: int
  currentLevelName: String
  pointsRequiredForCurrentLevel: int
  pointsToNextLevel: double
  levels: List<LevelRecord>    // all 8, ordered by level index
```

### `watchProgressionSummary()` → Stream<ProgressionSummary>

Live stream. Emits whenever profile or any level record changes.

### `getCurrentWeekProjection()` → WeekProjection

Reads current week score from `PerformanceService.getOverallWeekScore()` and determines which cup (if any) the user is on track to earn. Returns projected cup level and projected points.

```
WeekProjection
  projectedCupLevel: CupLevel?    // null if below bronze threshold
  projectedPoints: double
```

### `isReferralUnlocked()` → bool

Returns true if `currentLevel >= 2` and user is Pro/Premium.

---

## Rules

- The only writer of `ProgressionProfile` and `LevelRecord`
- `ProgressionService` knows only points — never reads `AchievementRecord` directly
- Carry-over on level-up — excess points transfer to next level
- All eight `LevelRecord` entries created at initialization
- `LevelReachedEvent` published after write is confirmed
- Max level (Penelope) never publishes `LevelReachedEvent` — profile continues accumulating points but no further level-up occurs

---

## Dependencies

- `ScoringService` — subscribes to `onPointsAwarded` (internal stream)
- `ProgressionRepository` — reads and writes `ProgressionProfile` and `LevelRecord`
- `PerformanceService.getOverallWeekScore()` — current week projection only
- `AppConfig` — `levelThresholds`