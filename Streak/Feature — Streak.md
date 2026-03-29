**File Name**: feature_streak **Phase**: 2 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Streak tracks consecutive kept and missed windows for each commitment using a single signed integer. It is the momentum signal of the app — used by the garment to accelerate or decelerate growth, and by the achievement system to recognize sustained consistency.

---

## Why It Exists

A streak is not just a count of good days. It is a measure of momentum — the direction and force of the user's recent pattern. A user on a 10-day kept streak is in a completely different state than a user who just broke their streak. A user on a -5 negative streak is struggling in a way that a single missed day does not capture.

Streak is a separate feature because multiple features above it need this momentum signal for different purposes — the garment uses it to adjust growth speed, the achievement system uses it to recognize milestones, the encouragement system uses it to calibrate feedback. Making it standalone means each consumer calls down into it independently without any knowledge of the others.

---

## Position in the System

Sits above Performance, Commitment, and TemporalHelper. Depends on Performance to classify windows as success or failure, Commitment for window close events and instance data, and TemporalHelper for the week-end signal that triggers weekly commitment evaluation. Below Garment and Achievements — both call `getStreakRecord()` downward.

Producing and consuming — it subscribes to events from below and publishes nothing. No `StreakChangedEvent` exists. External features that need streak data call `getStreakRecord()` directly.

---

## How It Works

Every commitment has one `StreakRecord` — created lazily on first window evaluation. The record holds a single signed integer (`currentStreak`) and the all-time best positive value (`bestStreak`).

**The signed streak.** Positive means consecutive kept windows. Negative means consecutive missed windows. Zero is neutral — either the commitment just started, or a streak just broke. The transition rules are simple:

- A kept window on a positive or zero streak increments by 1
- A missed window on a positive streak resets to 0 (not immediately negative — one missed day is not yet a bad streak)
- A missed window on a zero or negative streak decrements by 1
- A kept window on a negative streak resets to 0

This single value carries both the direction and the magnitude of momentum. It eliminates two separate counters and makes all transition logic trivial.

**Frozen windows are neutral.** A freeze does not break a streak or start a negative one. The record is left exactly as it was. The streak resumes from where it paused when the commitment is unfrozen.

**Weekly vs daily evaluation.** Daily and specific-day commitments are evaluated when their window closes. Weekly commitments are evaluated at week end — evaluating on daily window close would be wrong, since missing Tuesday does not mean the weekly goal failed. The week is the right unit for weekly commitments, and `StreakService` checks the instance's recurrence type at window close to skip weekly commitments there.

**Achievement detection.** After every streak update, the service checks two conditions internally — no external milestone feature needed:

- **Streak milestones** — when `currentStreak` crosses a configured threshold (3, 5, 7, 10, 14 days) for the first time, an achievement fires.
- **Global best** — when `bestStreak` reaches a new all-time high at a milestone value, a global best achievement fires and the `GlobalBestStreakRecord` is updated.

Global best is only published at milestone values — not on every new best. Without this gate, a user on a long streak would earn a new personal best achievement every single day from their previous high onward, diluting every one of them.

---

## Events

Streak publishes no events. No `StreakChangedEvent` exists — it was removed when the Milestones feature was eliminated. Milestone detection now lives inside this service where it already has all the data it needs. Any feature that needs streak data calls `getStreakRecord()` directly.

---

## Who Uses It

Any feature that needs to know the current momentum of a commitment calls `getStreakRecord()`. The garment acceleration feature reads it to adjust the multiplier. Any feature that needs to react to streak changes watches `watchStreakRecord()` for live updates on the detail screen.

---

## Rules

- No `StreakChangedEvent` — streak changes are internal
- Weekly commitments evaluated at week end only — not on daily window close
- Frozen windows leave the record unchanged — neither increment nor decrement
- `bestStreak` tracks positive peaks only — never affected by negative streaks
- Milestone achievements only at values in `AppConfig.streakMilestones`
- Global best achievement only when new best crosses a milestone value — prevents achievement dilution
- `commitmentName` read from instance snapshot — no definition lookup needed

---

## Later Improvements

No structural changes planned. Milestone thresholds are configurable via `AppConfig.streakMilestones` — tunable after launch without code changes.