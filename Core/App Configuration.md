**File Name**: appconfiguration **Feature**: Core **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 24-Mar-2026

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

commitmentAccessPerformanceThreshold: 0.50
  Minimum average performance required to add a new commitment (0.0–1.0).

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

successThreshold: 0.0
  Minimum livePerformance for a window to count as a kept day.
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
minStreakDays: 3
  Minimum absolute streak value (positive or negative) before
  StreakChangedEvent is published. Below this threshold the record
  updates silently. Prevents noise from single-day fluctuations.
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
cupThresholdBronze:  0.60
cupThresholdSilver:  0.75
cupThresholdGold:    0.85
cupThresholdDiamond: 0.95
  Weekly overall score thresholds for each cup level.
  Below cupThresholdBronze — no cup awarded that week.
```

---

## Achievements — Milestones

```
streakMilestones: [3, 5, 7, 10, 14]
  Positive streak counts that trigger a MilestoneRecord.
  Only positive streaks qualify — negative streaks have no milestones.
```

---

## Achievements — Rewards

```
rewardIntervalDays: 30
  Minimum days between occasional reward evaluations.
  Both this interval AND the performance threshold must be met.

rewardPerformanceThreshold: 0.65
  Minimum average performance over the reward interval to qualify.
```

---

## Progression — Points

```
cupPointsBronze:  1.0
cupPointsSilver:  2.0
cupPointsGold:    3.0
cupPointsDiamond: 5.0
  Points awarded per cup level. Read by ScoringService.

milestonePoints: 1.0
  Points per streak milestone. Flat — recognition is the reward,
  not a scaling bonus.

rewardPoints: 2.0
  Points per occasional reward.
```

---

## Progression — Bonus Triggers

```
bonusPointsFirstCup:          1.0
bonusPointsFirstDiamond:      1.0
bonusPointsThreeConsecutive:  1.0
bonusPointsComeback:          1.0
bonusPointsDiamondComeback:   2.0
  Bonus points for special cup pattern achievements.
  Evaluated by ScoringService on each CupEarnedInternalEvent.
  Priority order: DiamondComeback > FirstDiamond > FirstCup >
                  Comeback > ThreeConsecutive.
  Only the first match fires per evaluation.

bonusAwardMaxPerMonth: 1
  Maximum bonus awards per calendar month.
  Enforced by ScoringService via ScoringState.lastBonusAwardedMonth.
```

---

## Progression — Levels

```
levelThresholds: [0, 5, 15, 25, 40, 60, 80, 105]
  Cumulative points to reach each level. Index = level number.
  Level 0 = Apprentice (0 pts), Level 7 = Penelope (105 pts).
  See language_guide for level names and level-up messages.
```

---

## Grace and Re-notification

```
gracePeriodMinutes: 15
  Default grace duration after an activity window closes with no log.
  Users can override in UserDefaultPreferences. Zero disables grace.

renotificationMaxCount: 1
  Maximum re-notifications per instance window. Prevents repeated nudging.
```

---

## Scheduler

```
longIntervalMinutes: 15
  Tick interval for operations that tolerate delay —
  window closure evaluation, catch-up instance generation,
  day boundary detection.

shortIntervalMinutes: 1
  Tick interval for time-sensitive operations —
  warning notification checks, grace expiry checks.
  Phase 1: same as long interval. Separated for future tuning.
```

---

## AI Insights (Phase 3)

```
freeInsightQuotaPerWeek: 3
  Weekly micro-insight quota for free tier users.
  Checked and decremented by UserSettingsService.checkAndDecrementInsightQuota().

deepReportMinIntervalDays: 7
  Minimum days before a deep report can be regenerated.
  Enforced by the weekly summary component before calling AIInsightService.
```

---

## Rules

- All constants are compile-time — no runtime loading
- No service hardcodes any of these values — always reference AppConfig
- Changing a value affects all features that use it — test accordingly
- System constants only — not exposed to users in settings
- User-configurable defaults live in `UserDefaultPreferences`