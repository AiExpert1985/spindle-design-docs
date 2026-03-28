**File Name**: appconfiguration **Feature**: Core **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** single source of truth for all system-level constants. No service, repository, or component hardcodes these values — they always reference this class. Changing a value here changes it everywhere.

`AppConfig` is a pure class with static constants. No dependencies, no runtime loading, no database. Compile-time only.

User-configurable preferences (week start day, notification times, celebration threshold, etc.) live in `UserDefaultPreferences` — not here.

---

## Commitment Access

```
commitmentRateLimitWindowDays: 7
  Rolling window in days for rate limiting new commitment creation.

commitmentRateLimitMax: 3
  Maximum new commitments allowed within the rate limit window.

commitmentAccessPerformanceThreshold: 50
  Minimum average performance (0–100) required to add a new commitment.
  Consistent with PerformanceService percentage return values.

commitmentAccessPerformanceWindowDays: 7
  Days used to evaluate recent performance for access gating.

freePortfolioLimit: 7
  Maximum active commitments for free tier users.
  Pro and Premium have no ceiling.

onboardingBypassCount: 3
  Commitments created during onboarding that bypass all access gates.
```

---

## Instance and Logging

```
minimumWarningLeadMinutes: 15
  Minimum time remaining in the activity window for a warning notification
  to be useful. Warnings skipped if less than this remains.

maxLogBackfillDays: 7
  How many days in the past a user may attribute a log entry to.
  Covers genuine forgetfulness without enabling manipulation.
  Future dates never accepted.
```

---

## Scoring

```
avoidPenaltyBase: 0.5
  Controls how aggressively the score degrades when an avoid commitment
  exceeds its target. At 0.5, every doubling of excess halves the score.
  Lower = harsher, higher = softer.
  Only applies when totalLogged > target.value and target.value > 0.

successThreshold: 0
  Minimum livePerformance (0–100) for a window to count as a kept day.
  Used exclusively by PerformanceService.isWindowSuccess() — no feature
  reads this directly. Any progress counts in Phase 1 and 2.
  Phase 3: configurable per commitment via UI. This is the global fallback.
```

---

## Garment

```
garmentFloor:   0.0
garmentCeiling: 100.0
  Hard bounds for garment completion percentage.

targetRepetitions: 30
  How many successful windows to complete the garment (reach 100%).
  All recurrence types use the same value — weekly commitments are
  decomposed into daily instances, so every instance is daily.
```

---

## Streak

```
minStreakThreshold: 3
  Minimum absolute streak value (positive or negative) before
  achievement detection runs in StreakService._detectAchievements().
  Below this threshold the record updates silently.
  Prevents noise from single-window fluctuations.

streakMilestones: [3, 5, 7, 10, 14]
  Positive streak counts that trigger a streak milestone achievement.
  Only positive streaks qualify — negative streaks have no milestones.
  Matches the badge mapping in component_streak_ui.
  Adjustable after launch — badge mapping must be updated to match.
```

---

## Accelerator

```
garmentUsesAccelerator: true
  Master switch for the streak accelerator. When false, GarmentService
  uses raw livePerformance directly and AcceleratorService has no effect.

acceleratorFloor: 0.75
  Minimum multiplier value. Garment growth never slows below this.

acceleratorCeiling: 1.50
  Maximum multiplier value. Asymmetric with floor — momentum builds
  higher than it falls, reflecting that consistent repetition compounds
  habit formation faster than inconsistency degrades it.
```

---

## Achievements — Cups

```
cupThresholdBronze:  60
cupThresholdSilver:  75
cupThresholdGold:    85
cupThresholdDiamond: 95
  Weekly overall score thresholds (0–100) for each cup level.
  Consistent with PerformanceService.getOverallWeekScore() return values.
  Below cupThresholdBronze — no cup awarded that week.
```

---

## Progression — Points

All values are placeholders. They will be carefully designed after launch based on real user earning rates and desired progression pace. Only AppConfig needs to change — ScoringService and ProgressionService are untouched.

```
// Cup achievements
pointsBronzeCup:  1.0
pointsSilverCup:  2.0
pointsGoldCup:    3.0
pointsDiamondCup: 5.0

// Streak achievements
pointsGlobalBestStreak: 3.0

// Streak milestone achievements
pointsThreeDay:    1.0
pointsFiveDay:     1.0
pointsSevenDay:    2.0
pointsTenDay:      2.0
pointsFourteenDay: 3.0

// Garment achievements
pointsGarmentCompleted: 5.0

// Reward achievements — future feature, placeholder kept so ScoringService
// is ready when Rewards ships without any code changes needed
pointsPeriodicReward: 2.0
```

---

## Progression — Levels

```
levelThresholds: [0, 5, 15, 25, 40, 60, 80, 105]
  Points required to complete each level. Index = level number.
  Level 0 = Apprentice, Level 7 = Penelope.
  These are placeholders — will be tuned after launch based on
  real user earning rates. See language_guide for level names and copy.
```

---

## Scheduler

```
longIntervalMinutes: 15
  Tick interval for operations that tolerate delay —
  window closure evaluation, catch-up instance generation,
  boundary event detection.

shortIntervalMinutes: 1
  Tick interval for time-sensitive operations —
  warning notification checks.
  Phase 1: same as long interval. Separated for future tuning.
```

---

## AI Insights (Phase 3)

```
freeInsightQuotaPerWeek: 3
  Weekly micro-insight quota for free tier users.
  Checked and decremented by UserSettingsService.checkAndDecrementInsightQuota().
  Called only by AIInsightService — never by the presentation layer.

deepReportMinIntervalDays: 7
  Minimum days before a deep report can be regenerated.
  Enforced by AIInsightService.generateDeepReport() — not by the component.
```

---

## Cup Backfill

```
cupBackfillWeeks: 8
  How many past weeks CupService checks on startup for missing cup records.
  Bounded to prevent unbounded reads. Consistent with the ScoringService
  bonus trigger evaluation window.
```

---

## Rules

- All constants are compile-time — no runtime loading
- No service hardcodes any of these values — always reference AppConfig
- Changing a value affects all features that use it — test accordingly
- System constants only — not exposed to users in settings
- User-configurable defaults live in `UserDefaultPreferences`
- Point values are placeholders — see Progression section note