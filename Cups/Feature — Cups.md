**File Name**: feature_cups **Phase**: 2 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Cups recognize weekly performance across all commitments. At the end of each week, the user's overall score is evaluated and — if it meets a threshold — a cup is awarded at the appropriate level. Cups are the weekly milestone of the app: a cross-commitment signal that the user showed up consistently across the whole week.

---

## Why It Exists

Daily performance is already tracked per commitment. Cups add a second dimension — the weekly picture across everything the user is working on. A bronze cup says "you had a decent week overall." A diamond cup says "you kept everything this week." It rewards the user for breadth of consistency, not just depth on any single commitment.

Cups are also the simplest achievable win in the app. A user who had a rough week on some commitments but stayed above 60% overall still gets a bronze — an honest recognition that something was kept, even if imperfectly.

---

## Position in the System

Sits at the same level as Streak and Analytics — above Performance and TemporalHelper, below Garment and Progression. Depends on Performance for the weekly score and TemporalHelper for the week-end signal. Calls Achievements below it to record the cup. No feature at the same level depends on it.

Consuming only — subscribes to `WeekEndedEvent`, publishes nothing.

---

## How It Works

When a week ends, `CupService` reads the overall weekly performance score. If it meets one of four configurable thresholds, a cup is written and an achievement recorded. If the score falls below the minimum threshold, nothing happens — no record, no noise.

**Four levels — bronze, silver, gold, diamond** — with percentage thresholds configurable in `AppConfig` (defaults: 60%, 75%, 85%, 95%). The level reflects the quality of the week honestly and without inflation.

**One cup per week maximum.** The service checks before writing — if a cup already exists for that week, it exits silently. This makes the handler safe to call multiple times (e.g. on app startup catch-up) without risk of duplicates.

**Backfill on startup.** When the app opens, `CupService` looks back a configurable number of weeks and silently fills any missing cup records. This handles the case where the app was closed when a week ended. Backfilled cups are stored as historical records but do not fire achievements — they are not celebratory moments, just accurate history.

**Cups are cross-commitment.** `definitionId` is null on the achievement record — a cup belongs to the week, not to any specific commitment. This is what makes cups distinct from streak achievements, which are per-commitment.

**Raw performance only.** Cups are based on `getOverallWeekScore()` — the raw average of all instance `livePerformance` values for the week. The accelerator multiplier, which affects garment growth, never touches cup evaluation. A cup reflects what the user actually did, not a momentum-adjusted version of it.

---

## Events

Cups publishes no events. It consumes `WeekEndedEvent` from TemporalHelper and writes directly to Achievements via `AchievementService.addAchievement()`. The achievement system publishes `AchievementEarnedEvent` which any feature above can subscribe to.

---

## Rules

- One cup per week — idempotency checked before every write
- Raw performance score only — accelerator never applied to cup evaluation
- No achievement recorded for backfilled cups — historical records only
- `definitionId` is null on cup achievements — cups are cross-commitment
- Thresholds are configurable percentage values in `AppConfig`
- Below minimum threshold (default: 60%) — no cup, no record, no noise

---

## Later Improvements

No structural changes planned. Thresholds are tunable from `AppConfig` after launch without code changes.