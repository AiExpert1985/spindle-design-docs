**File Name**: component_streak_ui **Feature**: Streak **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 30-Mar-2026

---

**Purpose:** two distinct UI surfaces for streak data — a milestone overlay that fires when a milestone is reached, and a streak summary row used on the Your Record screen. Both read from `AchievementService`. No logic — purely presentational.

Expected placement: milestone overlay fires on any screen when a milestone is earned. Streak summary row appears on the Your Record screen, one row per active commitment.

---

## Surface 1 — Milestone Overlay

Fires when `AchievementEarnedEvent` arrives with `type: streakMilestone`. Full-screen overlay, auto-dismisses after ~2.5 seconds. Tap to dismiss early.

```
┌─────────────────────────────────────────┐
│                                         │
│                  🥇                     │
│            Morning Walk                 │
│          7 days in a row.               │
│             Gold streak.                │
│                                         │
└─────────────────────────────────────────┘
```

- Badge derived from `streakCount` via the badge mapping below
- Commitment name from `AchievementRecord.definitionId` → `AchievementService.getStreakRecord()`
- Auto-dismisses after ~2.5 seconds, tap dismisses immediately
- Triggered by `EncouragementService` — this component is the visual, Encouragement is the trigger

**Badge mapping** — driven by `AppConfig.streakStep` (default: 3). Badge tier advances every `streakStep` milestones:

|Streak count|Badge|
|---|---|
|1× streakStep (3)|🥉 Bronze|
|2× streakStep (6)|🥈 Silver|
|3× streakStep (9)|🥇 Gold|
|4× streakStep (12)|🏆 Trophy|
|5× streakStep (15)+|💎 Diamond|

Badge tier is `ceil(currentStreak / streakStep)`, capped at Diamond. Milestones beyond 5× streakStep continue earning achievements but keep the Diamond badge. Changing `streakStep` in AppConfig adjusts all thresholds automatically — the badge mapping derives from it.

---

## Surface 2 — Streak Summary Row (Your Record)

One row per active commitment showing current streak and personal best.

```
Morning Walk   7 days 🥇   Best: 12 days
Read Daily     3 days 🥉   Best: 21 days
No Alcohol    14 days 💎   Best: 14 days  ✨ Personal best!
```

- Current streak displayed as absolute value — never shows negative numbers to the user. A negative streak is shown as "— no current streak" with no shame, no red color.
- Personal best matched → "✨ Personal best!" in a warm accent color
- Frozen commitment → ❄ in place of streak count, no number shown

---

## Data Sources

|Data|Source|
|---|---|
|Streak record per commitment|`AchievementService.getStreakRecord(definitionId)`|
|Active commitments list|`CommitmentService.watchActiveCommitments()` — stream|
|Milestone overlay trigger|`AchievementEarnedEvent` where `type: streakMilestone` — via Encouragement|

The component receives data from the parent screen's provider — it does not subscribe to events directly. Event handling belongs in Encouragement.

---

## Rules

- Purely presentational — no service calls from within the component
- Negative `currentStreak` values are never shown as negative — shown as no current streak
- Milestone overlay fires once per milestone event — deduplication handled by `StreakService`
- Frozen commitments show ❄ state — no streak count, no badge
- Badge tier derived from `AppConfig.streakStep` — no separate mapping to maintain

---

## Later Improvements

**Streak history.** A tap on a commitment's streak row opens a history of past streak milestones for that commitment. Requires `AchievementService.getAchievements(type: streakMilestone, definitionId)`.

**Accelerator indicator.** A subtle visual showing current momentum state alongside the streak. Deferred until UX for communicating the accelerator concept is settled.