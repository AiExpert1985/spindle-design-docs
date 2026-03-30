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

Progression is composed of two internal services — `ScoringService` and `ProgressionService` — connected by an internal event. No feature outside Progression interacts with either service directly except through the public interface of `ProgressionService`.

### Scoring

`ScoringService` subscribes to `AchievementEarnedEvent`. When an achievement arrives, it looks up the point value for that subtype in `AppConfig` and calls `ProgressionService.awardPoints()` directly. That is all it does — translate achievement subtypes into point values and hand them off.

The lookup is `subtype → points`, not `type → points`, because different achievements of the same type have different values — a diamond cup is worth more than a bronze cup. The point values are all configurable placeholders, to be tuned from real user data after launch.

`ProgressionService` knows nothing about achievements. It only receives points. This means changing what achievements are worth, or adding new ones, never touches `ProgressionService`.

### Level Progression

`ProgressionService` subscribes to `PointsAwardedEvent`. It adds points to the current level's progress and checks for a level-up. **Points carry over on level-up** — if a level requires 10 points and the user earns 13, they start the next level with 3. This is intentional: excess points represent real effort and should not be erased. It also prevents a cliff-edge feeling when a burst of activity crosses a level boundary.

There are eight levels. All eight `LevelRecord` entries are created at account initialization — the user can see the full path from day one. Pending levels show their names and thresholds but have zero progress. When the user reaches the final level (Penelope), points continue accumulating but no further level-up fires.

### Live Level Map

The Progression screen shows all eight levels with live progress. Level records are maintained live on every points award — not computed on read. This means the screen is always a direct read with no recalculation. The cost is extra writes; the benefit is a screen that never lags behind reality.

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