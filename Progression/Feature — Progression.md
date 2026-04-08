**File Name**: feature_progression **Phase**: 2 (Scoring) · 3 (Progression visuals) **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Progression translates every achievement the user earns into points, accumulates those points toward levels, and tracks the user's journey through eight named levels from Apprentice to Penelope. It makes the long-term arc of building habits visible — not just today's streak or this week's cup, but where the user stands in their overall story.

---

## Why It Exists

Individual achievements — a cup, a streak milestone, a completed garment — are meaningful in isolation. But the user also needs to feel a larger arc of growth. Progression provides that. It answers the question: "How far have I come, and how far do I have to go?" It gives the user a reason to keep engaging even when no single commitment is crossing a threshold today.

The level names are drawn from the weaving world — Apprentice, Weaver, Journeyman, Artisan, Craftsman, Master Weaver, Loom Keeper, Penelope — consistent with the garment metaphor that runs through the app.

---

## Position in the System

Sits above Achievements, which it subscribes to for `AchievementEarnedEvent`. Also calls down into Performance for the weekly projection. Features that celebrate or notify on level-up sit above it and subscribe to `LevelReachedEvent`.

---

## How It Works

Progression is composed of two internal services — `ScoringService` and `ProgressionService` — connected by a direct call. No feature outside Progression interacts with either service directly except through the public interface of `ProgressionService`.

### Scoring

`ScoringService` subscribes to `AchievementEarnedEvent`. When an achievement arrives, it looks up the point value for that subtype in `AppConfig` and calls `ProgressionService.awardPoints()` directly.

**Why `ScoringService` is separate from `ProgressionService`.** `ProgressionService` only needs to know about points — it should never understand what an achievement is or what it is worth. Keeping scoring separate means changing point values, or adding new achievement subtypes, only touches `AppConfig` and the lookup table. `ProgressionService` is untouched in both cases.

**Why `subtype → points`, not `type → points`.** Different achievements of the same type have different values — a diamond cup is worth more than a bronze cup. Subtype is the specific semantic label, making it the right key. All values are configurable placeholders to be tuned after launch.

### Level Progression

`ProgressionService` receives points via `awardPoints()`, adds them to the current level's progress, and checks for a level-up.

**Why points carry over on level-up.** Resetting to zero on level-up would erase points the user legitimately earned. If a level requires 10 points and the user earns 13, they start the next level with 3 — the extra 3 represent real effort and should count. It also prevents a cliff-edge feeling when a burst of activity crosses a level boundary.

There are eight levels. All eight `LevelRecord` entries are created at account initialization — the user can see the full path from day one. Pending levels show their names and thresholds but have zero progress. When the user reaches the final level (Penelope), points continue accumulating but no further level-up fires.

### Live Level Map

The Progression screen shows all eight levels with live progress. Level records are maintained live on every points award — not computed on read.

**Why not compute on read.** The screen would need to re-read all achievement history and recalculate all points on every open. Maintaining live records means the screen is always a direct read with no recalculation. The cost is extra writes per points award; the benefit is a screen that always reflects current state instantly.

---

## Events

**Subscribes to:** `AchievementEarnedEvent` (via `ScoringService` internally).

**Publishes:** `LevelReachedEvent(newLevel, levelName, previousLevel)` — fired when the user crosses into a new level. Consumed by features above that celebrate or notify on level-up.

---

## Rules

- `ScoringService` calls `ProgressionService.awardPoints()` directly — no internal event needed
- `ProgressionService` knows only points — never reads `AchievementRecord` directly
- All point values in `AppConfig` — never hardcoded
- Points carry over on level-up — never reset to zero
- All eight `LevelRecord` entries created at initialization
- Exactly one `LevelRecord` has `status: current` at all times
- `LevelReachedEvent` not published at the final level — points accumulate silently
- All thresholds and point values are placeholders — tuned after launch

---

## Later Improvements

**Referral unlock.** Referral sharing becomes available at level 2 for Pro/Premium users. Already gated by `isReferralUnlocked()` in `ProgressionService`.

**Point value tuning.** All values are placeholders. After launch, design point values based on real user earning rates and desired progression pace. Only `AppConfig` changes — no code changes.


---

## Related Docs

[[Model — Level Record]]
[[Model — Progression Profile]]
[[Repository — Progression]]
[[Screen — Progression]]
[[Service — Progression]]
[[Service — Scoring]]