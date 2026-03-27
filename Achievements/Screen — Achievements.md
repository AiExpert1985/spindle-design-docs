**File Name**: screen_achievements **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** shows the user's complete achievement history in one scrollable list. A reflective screen — opened occasionally to review progress. Accessed from the Your Record screen or a notification.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  YOUR ACHIEVEMENTS                   │
│                                         │
│  [ All ] [ Cups ] [ Streaks ] [Garment] │
│          [ Milestones ] [ Rewards ]     │
│                                         │
│  💎  Diamond cup                        │
│      Week of Mar 17                     │
│                                         │
│  🥇  Morning Walk — 7 day streak        │
│      Mar 14                             │
│                                         │
│  🧣  Morning Walk — garment complete    │
│      Mar 12                             │
│                                         │
│  🥇  Gold cup · Week of Mar 10          │
│                                         │
│  🥉  Read Daily — 3 day streak          │
│      Feb 28                             │
│                                         │
│  🎁  Periodic reward · Feb 15           │
└─────────────────────────────────────────┘
```

---

## Filter Chips

Maps to `AchievementType` enum values. Tapping a chip calls: `AchievementService.getAchievements(from, to, type: selected)`

Default: All. Default period: last 90 days. Expands backward in 90-day increments on scroll.

---

## Achievement Rows

Each row: icon + label + date. Icon and label derived from `AchievementSubtype`. One display entry per subtype value — defined here. Tap → detail sheet.

**Cup:** "💎 Diamond cup · Week of Mar 17" **Milestone:** "🥇 Morning Walk — 7 day streak · Mar 14" **Global best streak:** "🏆 Morning Walk — new personal best · Mar 14" **Garment:** "🧣 Morning Walk — garment complete · Mar 12" **Reward:** "🎁 Periodic reward · Feb 15"

---

## Detail Sheet

Tap any row → bottom sheet. All detail data fetched via `sourceId` from `AchievementRecord` — calls `AchievementService` read functions only. No other feature calls needed.

**Cup detail:** level name, week, score from `AchievementService.getCupHistory()` **Streak detail:** streak count, current streak, best from `AchievementService.getStreakAchievements()` **Global best:** commitment name, days, date from `AchievementService.getBestStreak()` **Garment:** commitment name, date from `AchievementRecord` fields directly **Reward:** period, average score from `sourceId` lookup

---

## Navigation

Accessed from: Your Record screen · achievement notification tap. Back → previous screen.

---

## Data Sources

| Data             | Source                                                             |
| ---------------- | ------------------------------------------------------------------ |
| Achievement list | `AchievementService.getAchievements(from, to, type?)`              |
| Live updates     | `AchievementService.watchAchievements(from, to)` — stream          |
| Cup detail       | `AchievementService.getCupHistory(from, to)`                       |
| Streak detail    | `AchievementService.getStreakAchievements(definitionId, from, to)` |
| Global best      | `AchievementService.getBestStreak()`                               |