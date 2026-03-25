**File Name**: screen_achievements **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** shows the user's complete achievement history in one scrollable list. A reflective screen — opened occasionally to review progress. Accessed from the Your Record screen or a notification.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  YOUR ACHIEVEMENTS                   │
│                                         │
│  [ All ] [ Cups ] [ Streaks ] [ Rewards]│
│                                         │
│  💎  Diamond cup                        │
│      Week of Mar 17                     │
│                                         │
│  🥇  Morning Walk — 7 day streak        │
│      Mar 14                             │
│                                         │
│  🥇  Gold cup                           │
│      Week of Mar 10                     │
│                                         │
│  🥈  Silver cup                         │
│      Week of Mar 3                      │
│                                         │
│  🥉  Read Daily — 3 day streak          │
│      Feb 28                             │
│                                         │
│  🎁  30-day reward                      │
│      Feb 15                             │
└─────────────────────────────────────────┘
```

---

## Filter Chips

`[ All ] [ Cups ] [ Streaks ] [ Rewards ]` — All selected by default.

Maps to `AchievementType` enum values. Tapping a chip filters `AchievementService.getAchievements(type: selected)`.

---

## Achievement Rows

Each row shows: icon, label, date. One tap → detail sheet (see below).

**Cup row:** cup icon + level name + week label. Example: "💎 Diamond cup · Week of Mar 17"

**Streak milestone row:** milestone badge + commitment name + streak count. Example: "🥇 Morning Walk — 7 day streak · Mar 14"

**Reward row:** gift icon + reward type + date. Example: "🎁 30-day reward · Feb 15"

---

## Detail Sheet

Tap any row → bottom sheet with expanded detail.

**Cup detail:**

```
💎 Diamond cup
Week of Mar 17 · Score: 96%
You had an excellent week.
```

**Streak detail:**

```
🥇 7-day streak
Morning Walk · Mar 14
Current streak: 9 days
Best ever: 12 days
```

**Reward detail:**

```
🎁 30-day reward
Feb 15
Average score over 30 days: 74%
Keep weaving.
```

Detail data fetched via `sourceId` from the `AchievementRecord` — calls the appropriate internal-proxied read function on `AchievementService`.

---

## Navigation

- Accessed from: Your Record screen · achievement notification tap
- Back → returns to previous screen

---

## Data Sources

|Data|Source|
|---|---|
|Achievement list|`AchievementService.getAchievements(limit?, type?)`|
|Live updates|`AchievementService.watchRecentAchievements()` — stream|
|Cup detail|`AchievementService.getCupHistory()` filtered by sourceId|
|Streak detail|`AchievementService.getStreakRecord(definitionId)`|