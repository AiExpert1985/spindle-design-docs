**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Rewards **Phase**: 2 **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** tracks consecutive kept days per commitment and celebrates milestone moments with a full-screen overlay. The rarest and most meaningful immediate reward in the app. Encouragement only — no punishment when a streak breaks.

---

## Milestones

|Days in a row|Badge|
|---|---|
|3|🥉 Bronze|
|5|🥈 Silver|
|7|🥇 Gold|
|10|🏆 Trophy|
|14|💎 Diamond|

Streak resets silently on miss. Milestone celebrated again when reached next time.

---

## UI — Milestone Animation

Fires immediately when the user logs the activity that completes the milestone.

```
        🥇
   Morning Walk
7 days in a row.
   Gold streak.
```

- Full-screen overlay, ~2.5 seconds
- Badge animates in with subtle burst
- Auto-dismisses or tap to close
- Optional chime (respects silent mode)

The overlay is triggered by `EncouragementService` subscribing to `StreakMilestoneReachedEvent` and emitting a `MilestoneOverlaySignal`. The presentation layer watches this Riverpod signal.

---

## UI — Your Record (Section 3)

One row per active commitment:

```
Morning Walk   7 days 🥇   Best: 12d
Read Daily     3 days 🥉   Best: 21d
No Alcohol    14 days 💎   Best: 14d  ✨ Personal best!
```

- Current = 0 → `"— no current streak"` (no shame, no red)
- Personal best matched → `"✨ Personal best!"` in green
- Frozen commitment → ❄ in place of streak count

---

## Data Model

Streak data lives on the commitment definition:

```
currentStreak: int
bestStreak: int
bestStreakEndDate: DateTime?    // null if still running
```

---

## How It Works

`RewardService` subscribes to `ActivityRecordedEvent`. On each event:

1. Reads definition via `CommitmentService.getDefinition()`
2. Calls `updateStreak(definition, logResult)` — pure function
3. Saves updated definition via `CommitmentService.saveDefinitionState()`
4. If milestone reached → publishes `StreakMilestoneReachedEvent`

`EncouragementService` subscribes to `StreakMilestoneReachedEvent` → emits `MilestoneOverlaySignal` → presentation layer shows overlay.

**Freeze behavior:**

- Indefinite freeze → streak pauses at current count, resumes on unfreeze
- Dated freeze → freeze days are neutral, streak preserved through the period

---

## Dependencies

- `RewardService` — streak calculation, milestone detection, event publishing
- `EncouragementService` — subscribes to `StreakMilestoneReachedEvent`, emits overlay signal
- `EventBus` — event communication
- Presentation layer — watches `MilestoneOverlaySignal` Riverpod provider