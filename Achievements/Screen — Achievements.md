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
│                                         │
│  💎  Diamond cup                        │
│      Week of Mar 17                     │
│                                         │
│  🏆  New personal best — 14 days        │
│      Morning Walk · Mar 15              │
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
└─────────────────────────────────────────┘
```

---

## Filter Chips

Maps to `AchievementType` enum values. Tapping a chip calls `AchievementService.getAchievements(from, to, type: selected)`.

Current chips: All · Cups · Streaks · Garment.

Default: All. Default period: last 90 days. Expands backward in 90-day increments on scroll to the bottom.

---

## Achievement Rows

Each row: icon + label + date. Icon and label derived from `AchievementSubtype` — one display entry per subtype value, defined here. Tap → detail sheet.

|Subtype|Icon|Label format|
|---|---|---|
|bronzeCup|🥉|"Bronze cup · Week of [date]"|
|silverCup|🥈|"Silver cup · Week of [date]"|
|goldCup|🥇|"Gold cup · Week of [date]"|
|diamondCup|💎|"Diamond cup · Week of [date]"|
|threeDay|🥉|"[name] — 3 day streak"|
|fiveDay|🥈|"[name] — 5 day streak"|
|sevenDay|🥇|"[name] — 7 day streak"|
|tenDay|🏆|"[name] — 10 day streak"|
|fourteenDay|💎|"[name] — 14 day streak"|
|globalBestStreak|🏆|"New personal best — [n] days · [name]"|
|garmentCompleted|🧣|"[name] — garment complete"|

---

## Detail Sheet

Tap any row → bottom sheet. All data fetched from `AchievementService` — no other feature calls needed.

**Cup detail:**

```
💎 Diamond cup
Week of Mar 17 · Score: 96%
```

Data: `AchievementService.getCupHistory(from, to)` filtered by `sourceId`.

**Streak milestone detail:**

```
🥇 7-day streak
Morning Walk · Mar 14
Current streak: 9 days
Best ever: 14 days
```

Data: `AchievementService.getStreakAchievements(definitionId, from, to)`.

**Global best detail:**

```
🏆 New personal best — 14 days
Morning Walk · Mar 15
```

Data: `AchievementService.getBestStreak(definitionId)`.

**Garment detail:**

```
🧣 Garment complete
Morning Walk · Mar 12
```

Data: `AchievementRecord` fields directly — `definitionId` and `createdAt`.

---

## Navigation

Accessed from: Your Record screen · achievement notification tap. Back → previous screen.

---

## Data Sources

|Data|Source|
|---|---|
|Achievement list|`AchievementService.getAchievements(from, to, type?)`|
|Live updates|`AchievementService.watchAchievements(from, to)` — stream|
|Cup detail|`AchievementService.getCupHistory(from, to)`|
|Streak detail|`AchievementService.getStreakAchievements(definitionId, from, to)`|
|Global best detail|`AchievementService.getBestStreak(definitionId?)`|

---

## Later Improvements

**Rewards filter chip.** When the Rewards feature ships, a "Rewards" filter chip is added. `AchievementService` already stores reward records — the chip just calls `getAchievements(from, to, type: reward)`. No other change needed.