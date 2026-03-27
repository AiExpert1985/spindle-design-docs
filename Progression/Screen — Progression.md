**File Name**: screen_progression **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** shows the user's complete progression journey — current level, points toward next level, the full level map, cup breakdown, and current week projection. The reflective screen for long-term motivation.

Not in bottom nav. Accessed via Your Record screen header teaser · deep-link `/progression` · level-up notification tap.

Available to all tiers — progression tracking is free. Referral section visible to Pro/Premium only from level 2 onward.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  YOUR JOURNEY                        │
│                                         │
│  Artisan                                │
│  ●●●●●●●░░░░░░░  32 / 50 pts           │
│  18 points to Craftsman                 │
│                                         │
│  On track for Silver this week  +2 pts  │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  CUPS EARNED                            │
│  🥉×8   🥈×5   🥇×3   💎×1             │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  YOUR PATH                              │
│  ✓ Apprentice    started               │
│  ✓ Weaver        Mar 18                │
│  ✓ Journeyman    Apr 22                │
│  ✓ Artisan       Jun 4                 │
│  → Craftsman     ●●●░░  32/50 pts      │
│    Master Weaver  need 18 more          │
│    Loom Keeper                          │
│    Penelope                             │
│                                         │
│  [ What is this? ]                      │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  [ Share your achievement ]             │
│  Pro / Premium · level 2+ only          │
└─────────────────────────────────────────┘
```

---

## Sections

**Header — Current Level.** Level name in large text. Progress bar from 0 to `pointsRequired`. Points shown as `currentPoints / pointsRequired`. Points remaining below. At max level (Penelope): bar full, "You reached the final level."

**On Track This Week.** Live indicator. Shows projected cup and projected points. If below bronze threshold: "No cup on track this week" — neutral, no shame.

**Cups Earned.** Four cup icons with counts. Tap any icon → popup listing weeks earned at that level.

**Your Path.** All 8 levels visible. Done: checkmark + date. Current: arrow + mini progress bar + points remaining. Future: name only, greyed. Showing the full map motivates more than showing only the next step.

**What Is This?** Static explanation of cups, points, level table, and how achievements translate to points.

**Share Achievement.** Tap → Referral screen. Hidden for free users and users below level 2.

---

## Level-Up Celebration

When opened after a level-up (via notification), a brief overlay fires first:

```
┌─────────────────────────────────────────┐
│             ✦  Artisan  ✦               │
│   Consistent craft, recognizable work.  │
└─────────────────────────────────────────┘
```

Auto-dismisses after ~2.5 seconds. Fires only once per level — detected by checking today's date against `LevelRecord.achievedAt`.

---

## Navigation

- Back → previous screen
- `[ What is this? ]` → static explanation screen
- `[ Share your achievement ]` → Referral screen
- Deep-link: `/progression`

---

## Data Sources

| Data                               | Source                                                          |
| ---------------------------------- | --------------------------------------------------------------- |
| Full progression state + level map | `ProgressionService.watchProgressionSummary()` — stream         |
| Current week projection            | `ProgressionService.getCurrentWeekProjection()` — one-time read |
| Cup counts and history             | `AchievementService.getCupHistory(from, to)`                    |
| Referral unlock status             | `ProgressionService.isReferralUnlocked()`                       |