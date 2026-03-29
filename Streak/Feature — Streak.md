**File Name**: feature_streak **Phase**: 2 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Streak tracks consecutive kept and missed windows for each commitment using a single signed integer. It is the momentum signal of the app — used by the garment to accelerate or decelerate growth, and by the achievement system to recognize sustained consistency.

---

## Why It Exists

A streak is not just a count of good days. It is a measure of momentum — the direction and force of the user's recent pattern. A user on a 10-day kept streak is in a completely different state than a user who just broke one. A user on a -5 negative streak is struggling in a way a single missed day does not capture.

Streak is a separate feature because multiple features above it need this momentum signal for different purposes — the garment uses it to adjust growth speed, the achievement system uses it to recognize milestones, the encouragement system uses it to calibrate feedback. Keeping it standalone means each consumer calls down independently without knowing the others exist.

---

## Position in the System

Sits above Performance, Commitment, and TemporalHelper. Depends on Performance to classify windows as success or failure, Commitment for window close events and instance data, and TemporalHelper for the week-end signal that triggers weekly commitment evaluation. Below Garment and Achievements — both call `getStreakRecord()` downward.

Consuming only — subscribes to events from below, publishes nothing. No `StreakChangedEvent` exists. External features that need streak data call `getStreakRecord()` directly.

---

## How It Works

Every commitment has one `StreakRecord` — created lazily on first window evaluation. The record holds a single signed integer (`currentStreak`) and the all-time best positive value (`bestStreak`) with the date it was achieved.

**The signed streak.** Positive means consecutive kept windows. Negative means consecutive missed windows. Zero is neutral. The transition rules are simple:

```
Positive, window kept   → increment by 1
Positive, window missed → reset to 0        // one miss is neutral, not yet bad
Zero,     window kept   → increment to +1
Zero,     window missed → decrement to -1
Negative, window kept   → reset to 0        // one kept day clears the negative
Negative, window missed → decrement by 1
```

A single signed integer eliminates two separate counters and makes all transitions trivial. Zero as a neutral buffer between positive and negative is intentional — one missed day after a long good streak is not immediately a failure pattern. The magnitude of either direction is equally meaningful: a -7 streak signals sustained struggle just as clearly as a +7 streak signals sustained momentum.

**Frozen windows are neutral.** A freeze does not break a streak. The record is left unchanged — the streak resumes from where it paused.

**Weekly vs daily evaluation.** Daily and specific-day commitments are evaluated on window close. Weekly commitments are evaluated at week end — evaluating on daily window close would be wrong, since missing Tuesday on a weekly commitment does not mean the goal failed. The week is the right unit.

---

### No GlobalBestStreakRecord Model

There is no dedicated model or storage for the global best streak. It is computed on demand by scanning all `StreakRecord`s — at most a few dozen per user — and finding the highest `bestStreak`. This is cheaper and simpler than maintaining a separate document that must be kept in sync.

When a commitment's `bestStreak` updates, the service checks whether it now exceeds every other commitment's `bestStreak`. If it does — strictly greater, not tied — the global best achievement fires. A tie means another commitment already holds that value, so no new global best has been set.

---

### Achievement Detection

Two distinct achievements, two distinct meanings:

**Per-commitment milestone** — fires every time `currentStreak` is a positive multiple of 3 (3, 6, 9, 12...). Fires per commitment independently — each commitment runs its own streak, and each deserves recognition on its own terms. Re-earning after a break is intentional: rebuilding a streak to the same level is real effort and deserves acknowledgment again. Starting at 3 gives the user an early signal that something is forming without making it trivial.

Every 3 days (rather than a hand-picked list) is a formula, not a config array. It handles any streak length automatically — no milestone list to maintain, no badge mapping to keep in sync.

**Global best streak** — fires once, when this commitment's new `bestStreak` is strictly greater than every other commitment's `bestStreak`. This is the cross-commitment recognition — the user's single best performance across everything they track. Ties do not count — a tie means the global record already exists at that value.

These two achievements serve different purposes and never overlap. Per-commitment milestones recognize sustained effort on a specific commitment. Global best recognizes the user's best performance across their entire history.

---

## Events

Streak publishes no events. `StreakChangedEvent` was removed when the Milestones feature was eliminated — milestone detection now lives inside this service where it already has all the data it needs. No external feature needs to subscribe to streak changes.

---

## Rules

- No `StreakChangedEvent` — streak changes are internal
- Weekly commitments evaluated at week end only — not on daily window close
- Frozen windows leave the record unchanged
- `bestStreak` tracks positive peaks only — never affected by negative values
- Per-commitment milestone fires at every positive multiple of 3 — no config list needed
- Global best fires only when strictly greater than all other `bestStreak` values — ties ignored
- `GlobalBestStreakRecord` model does not exist — global best computed on demand from existing records
- `commitmentName` read from instance snapshot — no definition lookup needed

---

## Later Improvements

**`bestStreakDate`** on `StreakRecord` — records when the personal best was achieved, useful for the Your Record display. Deferred until the display screen is designed.