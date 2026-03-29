**File Name**: feature_streak **Phase**: 2 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

# Feature — Streak

Streak tracks consecutive kept and missed windows for each commitment using a single signed integer. It is the momentum signal of the app — consumed by features that need to know the direction and strength of a user's recent pattern.

---

## Why It Exists

A streak is not just a count of good days — it is a measure of momentum. A user on a +10 streak is in a completely different state than one who just broke theirs. A -5 streak signals sustained struggle just as clearly as +5 signals sustained momentum.

Streak is standalone because multiple features above it need this signal for different purposes. Each calls down independently without knowing the others exist.

---

## Position in the System

Sits above Performance, Commitment, and TemporalHelper. Depends on Performance to classify windows as success or failure, Commitment for window close events and instance data, and TemporalHelper for the week-end signal. Below Achievements — calls `AchievementService.addAchievement()` downward.

Consuming only — publishes nothing. No `StreakChangedEvent` exists. Features that need streak data call `getStreakRecord()` directly.

---

## How It Works

Every commitment has one `StreakRecord` — created lazily on first window evaluation. It holds a signed integer (`currentStreak`), the all-time best positive value (`bestStreakValue`), and the date that best was achieved (`bestStreakDate`). `bestStreakDate` is null until the first positive streak — set whenever `currentStreak` exceeds `bestStreakValue`.

**The signed streak.** Positive = consecutive kept windows. Negative = consecutive missed windows. Zero = neutral.

```
Positive, window kept   → increment by 1
Positive, window missed → reset to 0       // one miss is neutral, not yet a bad streak
Zero,     window kept   → increment to +1
Zero,     window missed → decrement to -1
Negative, window kept   → reset to 0       // one kept day clears the negative
Negative, window missed → decrement by 1
```

One number carries both direction and magnitude. Zero as a neutral buffer is intentional — one missed day after a long good streak should not immediately become a negative signal.

**Frozen windows are neutral.** The record is left unchanged — the streak resumes from where it paused.

**Weekly vs daily evaluation.** Daily and specific-day commitments are evaluated on window close. Weekly commitments are evaluated at week end — evaluating on daily close would be wrong, since missing one day on a weekly commitment does not mean the goal failed. The entire week must close before the streak can be judged. `StreakService` checks the instance's recurrence type at window close and skips weekly commitments there, processing them when `WeekEndedEvent` fires instead.

---

### No GlobalBestStreakRecord Model

The global best is computed on demand by scanning all `StreakRecord`s — at most a few dozen per user — and finding the highest `bestStreakValue`. No dedicated model or separate document to maintain. When a commitment's `bestStreakValue` updates, the service checks whether it is now strictly greater than every other. A tie does not count — it means the record already exists at that value.

---

### Achievement Detection

Two distinct achievements, two distinct meanings:

**Per-commitment milestone** — fires whenever `currentStreak` is a positive multiple of 3 (3, 6, 9, 12...). Per commitment, independently. Re-earning after a break is intentional — rebuilding a streak is real effort. A formula rather than a config list means it handles any streak length automatically with no maintenance.

**Global best** — fires when this commitment's `bestStreak` is strictly greater than every other commitment's `bestStreak`. Once. Cross-commitment recognition of the user's single best performance across everything they track.

The two never overlap — per-commitment milestones recognize individual effort, global best recognizes lifetime performance.

---

## Rules

- No `StreakChangedEvent` — streak changes are internal, features call `getStreakRecord()` directly
- Weekly commitments evaluated at week end only — recurrence type checked at window close
- Frozen windows leave the record unchanged
- `bestStreakValue` and `bestStreakDate` updated together whenever a new positive best is reached
- Per-commitment milestone fires at every positive multiple of `AppConfig.streakStep` (default: 3)
- Global best fires only when strictly greater than all other `bestStreakValue`s — ties ignored
- Global best computed on demand — no dedicated model