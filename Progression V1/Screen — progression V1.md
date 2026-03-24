
**Created**: 17-Mar-2026 
**Modified**: - 
**Feature**: Progression 
**Phase**: 3

**Availability:** all tiers — progression tracking is free. Referral section visible to Pro/Premium only from level 2 onward.

**Standalone:** yes — can be added without touching other screens.

**Access:** Your Record screen header teaser (tap) · deep-link `/progression` · level-up notification tap

**Purpose:** shows the user's complete progression journey — current level, points accumulated, what is needed to reach the next level, cup breakdown, bonus threads earned, and current week projection. The reflective screen for long-term motivation. Opened after Sunday calculation, after a level-up, or when the user wants to understand their standing.

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
│  [ Share your achievement ]  (lvl 2+)  │
│  Pro/Premium only                       │
└─────────────────────────────────────────┘
```

---

## Sections

### Header — Current Level

- Level name in large text
- Progress bar from current level threshold to next level threshold
- Points shown as `current / cumulative needed for next level`
- Points remaining stated plainly below the bar
- If at max level (Penelope): bar is full, no "points to next" shown. Text reads: `"You reached the final level."`

### On Track This Week

Live indicator. Updates as user logs commitments during the week.

```
On track for Silver this week  +2 pts
```

If below 60% (no cup on track): `"No cup on track this week"` — neutral, no shame.

Source: `ProgressionService.getCurrentWeekProjection()`

---

### Cups Earned

Four cup icons with counts. Tap any cup icon → same popup as Your Record showing list of weeks earned at that level.

Points breakdown shown as two lines:

```
32 pts from cups · 3 bonus threads
```

Bonus threads shown separately — never merged into a single number. Users should understand exactly where their points came from.

---

### Your Path

Full level table. All 7 levels visible at once.

- Completed levels: checkmark + date achieved
- Current level: arrow indicator
- Future levels: name only, greyed
- Next level: shows points still needed

This is the "full map" — users who can see the entire journey stay more motivated than users who only see the next step.

---

### What Is This?

Tapping opens a static explanation screen. One scroll. Covers:

- What cups are and how they are earned (threshold table)
- What progression points are and how cups translate to points (point value table)
- What bonus threads are and when they appear (trigger table, no specific numbers — just descriptions)
- Full level table with all names and cumulative points
- One-line flavor text per level

This screen is purely informational. No interaction needed. Back button returns to Progression Screen.

---

### Share Achievement (Phase 3 — Level 2+, Pro/Premium)

Single button. Tap → opens Referral Screen.

Hidden entirely for free tier users and for users below level 2. Not shown as disabled — just absent.

---

## Level-Up Celebration

When the user opens this screen immediately after a level-up (navigated from the level-up notification), a brief full-screen overlay fires first:

```
┌─────────────────────────────────────────┐
│                                         │
│             ✦  Artisan  ✦               │
│                                         │
│   Consistent craft, recognizable work.  │
│                                         │
└─────────────────────────────────────────┘
```

- Auto-dismisses after ~2.5 seconds
- Same pattern as Day Celebration — dark overlay, auto-dismiss, tap to skip
- Transitions into the normal Progression Screen underneath
- Only fires once per level — tracked via `levelAchievedDates` (if today's date matches the level date, show celebration)

---

## Navigation

- Back → returns to previous screen (Your Record or notification entry point)
- `[ What is this? ]` → static explanation screen
- `[ Share your achievement ]` → Referral Screen
- Deep-link: `/progression`

---

## Data Sources

|Data|Source|
|---|---|
|Full progression state|`ProgressionService.getProgressionSummary()`|
|Current week projection|`ProgressionService.getCurrentWeekProjection()`|
|Cup list for tap popup|`RewardService.fetchAllCups()`|
|Referral unlock status|`ProgressionService.isReferralUnlocked()`|