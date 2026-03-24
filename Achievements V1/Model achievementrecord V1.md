**Created**: 18-Mar-2026 **Modified**: - **Feature**: Achievements **Phase**: 2

**Purpose:** one record per significant positive event in the app. Append-only. The data source for the achievements feed. Written by `AchievementsService` in response to events from other features — those features know nothing about this model.

---

## Fields

```
id: String                    // client-generated UUID
occurredAt: DateTime          // when the achievement happened — from the source event
category: String              // see categories below — for filtering and display grouping
narrativeText: String         // ready-to-display feed line, assembled at write time
                              // e.g. "Morning Walk: 7-day Gold streak 🥇"
                              // e.g. "You reached Artisan — consistent craft, recognizable work."
                              // e.g. "Gold cup earned — week of Mar 10"
metadata: Map<String, dynamic> // category-specific data for rich display or future use
                              // not used for narrative — narrativeText is always the display
createdAt: DateTime           // when this record was written — may differ from occurredAt
                              // for backfilled records
```

---

## Categories

|Category|Source event|Example narrative|
|---|---|---|
|`first_thread`|`CommitmentCreatedEvent` (first ever)|"You spun your first thread: Morning Walk"|
|`thread_spun`|`CommitmentCreatedEvent`|"New thread spun: No alcohol"|
|`thread_completed`|`CommitmentCompletedEvent`|"Morning Walk — completed. The thread holds."|
|`streak_milestone`|`StreakMilestoneReachedEvent`|"Morning Walk: 7-day Gold streak 🥇"|
|`cup_earned`|`CupEarnedEvent`|"Gold cup earned — week of Mar 10"|
|`level_reached`|`LevelReachedEvent`|"You reached Artisan — consistent craft, recognizable work."|
|`bonus_thread`|`BonusThreadAwardedEvent`|"Bonus thread: comeback week +1"|

---

## Metadata Examples

```
// cup_earned
{ cupLevel: 'gold', weekStart: '2026-03-10', pointValue: 3.0 }

// streak_milestone
{ commitmentName: 'Morning Walk', streakCount: 7, badge: '🥇', definitionId: '...' }

// level_reached
{ newLevel: 3, levelName: 'Artisan', previousLevel: 2 }

// bonus_thread
{ bonusAmount: 1.0, trigger: 'comeback', weekStart: '2026-03-10' }

// thread_spun / first_thread
{ commitmentName: 'Morning Walk', commitmentType: 'do', definitionId: '...' }
```

---

## Append-Only Rule

Records are never edited or deleted. The feed is a permanent record of positive moments.

One exception: when a commitment is permanently deleted, its associated achievement records are soft-hidden (a `hiddenAt` timestamp) — they do not appear in the feed but are not erased. This preserves honesty — the achievement happened, even if the commitment no longer exists.

---

## Rules

- `narrativeText` is assembled at write time by `AchievementsService` — never derived at read time
- `metadata` is for rich display and analytics — not for reconstructing the narrative
- Negative entries (decay, missed windows) are never written — this feed is positive signal only
- One record per event — no deduplication needed since source events are already idempotent
- `occurredAt` uses the timestamp from the source event — not the write time