**File Name**: service_progression **Feature**: Progression **Phase**: 3 **Created**: 26-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** maintains the user's progression state. Receives points from `ScoringService` via `awardPoints()`, updates `currentPoints` and the `LevelRecord` collection, detects level-ups, and publishes `LevelReachedEvent`. The only writer of `ProgressionProfile` and `LevelRecord`.

---

## Public Write Function

### `awardPoints(points, subtype)` — called by ScoringService

```
profile = getOrCreateProfile()
currentLevel = getOrCreateCurrentLevel()

currentLevel.pointsEarned += points
profile.currentPoints += points

// check for level-up — loop handles multi-level skip from single large award
while currentLevel.pointsEarned >= currentLevel.pointsRequired
    and currentLevel.level < maxLevel:

  excessPoints = currentLevel.pointsEarned - currentLevel.pointsRequired
  currentLevel.pointsEarned = currentLevel.pointsRequired
  currentLevel.status = done
  currentLevel.achievedAt = now
  saveLevel(currentLevel)

  nextLevel = getLevelRecord(currentLevel.level + 1)
  nextLevel.status = current
  nextLevel.pointsEarned = excessPoints
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
saveLevel(currentLevel)
```

---

## Initialization

On first `awardPoints()` call:

1. Create `ProgressionProfile` with `currentPoints: 0`, `currentLevel: 0`
2. Create all eight `LevelRecord` entries — level 0 as `status: current`, levels 1–7 as `status: pending`

Lazy initialization — no separate setup step needed.

---

## Events Published

```
LevelReachedEvent
  newLevel: int
  levelName: String
  previousLevel: int
```

---

## Read Functions

### `getProgressionSummary()` → ProgressionSummary

One-time read. Assembles `ProgressionProfile` + all `LevelRecord` entries.

```
ProgressionSummary
  currentPoints: double
  currentLevel: int
  currentLevelName: String
  pointsRequiredForCurrentLevel: int
  pointsToNextLevel: double
  levels: List<LevelRecord>
```

### `watchProgressionSummary()` → Stream<ProgressionSummary>

Live stream. Emits whenever profile or any level record changes.

### `getCurrentWeekProjection()` → WeekProjection

```
WeekProjection
  projectedCupLevel: CupLevel?
  projectedPoints: double
```

Reads current week score from `PerformanceService.getOverallWeekScore()` and projects the cup the user is on track to earn.

### `isReferralUnlocked()` → bool

Returns true if `currentLevel >= 2` and user is Pro/Premium.

---

## Rules

- The only writer of `ProgressionProfile` and `LevelRecord`
- Never reads `AchievementRecord` — receives points only
- Carry-over on level-up — excess points transfer to next level
- All eight `LevelRecord` entries created at initialization
- `LevelReachedEvent` published after write is confirmed
- Final level (Penelope) never publishes `LevelReachedEvent` — points accumulate silently

---

## Dependencies

- `ScoringService` — calls `awardPoints()` directly
- `ProgressionRepository` — reads and writes `ProgressionProfile` and `LevelRecord`
- `PerformanceService.getOverallWeekScore()` — current week projection only
- `AppConfig` — `levelThresholds`