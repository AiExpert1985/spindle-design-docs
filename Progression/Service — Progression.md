**File Name**: screen_progression **Feature**: Core **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** shows the user's complete progression journey — current level, points accumulated, what is needed to reach the next level, cup breakdown, bonus threads earned, and current week projection. The reflective screen for long-term motivation. Opened after a level-up, after the Sunday calculation, or when the user wants to understand their standing.

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
│  32 pts from cups · 3 bonus threads     │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  YOUR PATH                              │
│  ✓ Apprentice    started               │
│  ✓ Weaver        Mar 18                │
│  ✓ Journeyman    Apr 22                │
│  ✓ Artisan       Jun 4                 │
│  → Craftsman     need 18 more pts      │
│    Master Weaver                        │
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

**Header — Current Level.** Level name in large text. Progress bar from current level threshold to next. Points shown as `current / cumulative needed for next level`. Points remaining stated plainly below the bar. At max level (Penelope): bar is full, text reads "You reached the final level."

**On Track This Week.** Live indicator — updates as the user logs during the week. Shows projected cup and points for the current week based on live performance. If below 60% (no cup on track): "No cup on track this week" — neutral, no shame.

**Cups Earned.** Four cup icons with counts. Tap any icon → popup showing list of weeks earned at that level. Points breakdown shown as two separate lines — cup points and bonus threads never merged into a single number.

**Your Path.** Full level table, all 8 levels visible at once. Completed levels show checkmark and date. Current level shows arrow indicator. Next level shows points still needed. Future levels greyed, name only. Showing the full map keeps users more motivated than showing only the next step.

**What Is This?** Tap → static explanation screen covering cups, points, bonus threads, and the full level table with flavor text. Read-only, back returns here.

**Share Achievement.** Single button. Tap → Referral screen. Hidden entirely for free users and users below level 2 — not shown as disabled, just absent.

---

## Level-Up Celebration

When the screen is opened immediately after a level-up (via level-up notification), a brief full-screen overlay fires first:

```
┌─────────────────────────────────────────┐
│                                         │
│             ✦  Artisan  ✦               │
│                                         │
│   Consistent craft, recognizable work.  │
│                                         │
└─────────────────────────────────────────┘
```

Auto-dismisses after ~2.5 seconds. Tap to skip. Transitions into the normal screen underneath. Fires only once per level — detected by checking whether today's date matches the level's entry in `levelAchievedDates`.

---

## Navigation

- Back → returns to previous screen
- `[ What is this? ]` → static explanation screen
- `[ Share your achievement ]` → Referral screen
- Deep-link: `/progression`

---

## Data Sources

|Data|Source|
|---|---|
|Full progression state|`ProgressionService.getProgressionSummary()` — one-time read|
|Current week projection|`ProgressionService.getCurrentWeekProjection()` — one-time read|
|Cup list for tap popup|`AchievementService.getCupHistory()`|
|Referral unlock status|`ProgressionService.isReferralUnlocked()`|