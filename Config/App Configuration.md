**File Name**: appconfiguration **Feature**: Config **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** single source of truth for all system-level constants. No service, repository, or component hardcodes these values — they always reference this class. Changing a value here changes it everywhere.

`AppConfig` is a pure class with static constants. No dependencies, no runtime loading, no database. Compile-time only.

---

## Commitment Access

```
commitmentRateLimitWindowDays: 7
  Rolling window in days for rate limiting new commitment creation.

commitmentRateLimitMax: 3
  Maximum new commitments allowed within the rate limit window.

commitmentAccessPerformanceThreshold: 0.50
  Minimum average performance ratio required to add a new commitment (0.0–1.0).

commitmentAccessPerformanceWindowDays: 7
  Days used to evaluate recent performance for access gating.

freePortfolioLimit: 7
  Maximum active commitments for free tier users. Pro and Premium have no ceiling.

onboardingBypassCount: 3
  Initial commitments that bypass all access gates during onboarding.
```

---

## Instance and Notification Timing

```
minimumWarningLeadMinutes: 15
  Minimum time remaining in the activity window for a warning notification
  to be useful. Warnings are skipped if less than this remains.

maxLogBackfillDays: 7
  How many days in the past a user may attribute a log entry to.
  Covers genuine forgetfulness (travel, illness) while limiting manipulation.
  Future dates are never accepted regardless of this value.
```

## Scoring

```
avoidPenaltyBase: 0.5
  Controls how aggressively the score degrades when an avoid commitment
  exceeds its target. At 0.5, every doubling of the excess halves the score.
  Lower values are harsher, higher values are softer.
  Only applies when totalLogged > target.value and target.value > 0.
```

Instance generation no longer uses a lead-time fraction. Instances are self-replicating — each instance spawns its successor when it closes. No tick-based lookahead is needed. The only timing constant needed is the minimum warning lead time for notifications.

---

## Scoring

```
successThreshold: 0.0
  Minimum livePerformance for a window to count as a success.
  Phase 1 and 2: always 0.0 — any progress counts.
  Phase 3: configurable per commitment via UI. This is the global fallback.

performanceDecayThreshold: 2
  Consecutive windows below successThreshold before garment decay activates.

performanceDecayAmount: 1.0
  Garment completion percentage points removed per window once decay activates.

performanceStreakBonusThreshold: 3
  Consecutive successful windows before streak bonus activates.

performanceStreakBonusAmount: 1.0
  Garment completion percentage points added per window once streak bonus activates.

targetRepetitions: 30
  How many successful windows to complete the garment (reach 100%).
  All recurrence types use the same value — weekly commitments are decomposed
  into daily instances, so every instance is a daily instance regardless of
  the original recurrence type. One constant covers all cases.
```

---

## Grace and Re-notification

```
gracePeriodMinutes: 15
  Default grace duration after an activity window closes with no log.
  Users can override in settings. Zero disables grace.

renotificationMaxCount: 1
  Maximum re-notifications per instance window. Prevents repeated nudging.
```

---

## Weekly Cups

```
cupThresholdBronze:  0.60
cupThresholdSilver:  0.75
cupThresholdGold:    0.85
cupThresholdDiamond: 0.95
  Weekly overall score thresholds for each cup level.

cupPointsBronze:  1.0
cupPointsSilver:  2.0
cupPointsGold:    3.0
cupPointsDiamond: 5.0
  Progression points awarded per cup.
```

---

## Progression Levels

```
levelThresholds: [0, 5, 15, 25, 40, 60, 80, 105]
  Cumulative cup points to reach each level. Index = level number.

bonusThreadFirstCup:          1.0
bonusThreadFirstDiamond:      1.0
bonusThreadThreeConsecutive:  1.0
bonusThreadComeback:          1.0
bonusThreadDiamondComeback:   2.0
  Bonus thread amounts for milestone achievements.

bonusAwardMaxPerMonth: 1
  Maximum bonus thread awards per calendar month per user.
```

---

## Thread Rope (Phase 3)

```
ropeDoStartingThreads:    3
ropeAvoidStartingThreads: 21
ropeMissGracePeriod:      2
  Consecutive missed windows allowed before thread penalty applies.
```

---

## Scheduler

```
longIntervalMinutes: 15
  Tick interval for operations that can tolerate delay —
  window closure evaluation, catch-up instance generation, day boundary detection.

shortIntervalMinutes: 1
  Tick interval for time-sensitive operations —
  warning notification checks, grace expiry checks.
  Phase 1: same as long interval. Separated for future tuning.
```

---

## AI Insights (Phase 3)

```
freeInsightQuotaPerWeek:   3
  Weekly micro-insight quota for free tier users.

deepReportMinIntervalDays: 7
  Minimum days before a deep insight report can be regenerated.
```

---

## Garment

```
garmentFloor:   0.0
garmentCeiling: 100.0
  Hard bounds for garment completion percentage.
```

---

## Rules

- All constants are compile-time — no runtime loading
- No service hardcodes any of these values
- Changing a value affects all features that use it — test accordingly
- These are system constants — not exposed to users in settings
- User-configurable defaults live in userdefaultpreferences