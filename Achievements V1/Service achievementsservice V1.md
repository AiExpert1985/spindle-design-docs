**Created**: 18-Mar-2026 **Modified**: - **Feature**: Achievements **Phase**: 3

**Purpose:** the single subscriber that converts significant positive events into permanent achievement records. Contributing features publish events and know nothing about this service. `AchievementsService` subscribes, assembles narrative text from the event payload, and writes one `AchievementRecord` per event. Provides the feed for display.

---

## Design Principle

Every event that feeds into achievements contains all the data needed to write a complete record. `AchievementsService` never calls another feature's service to fetch additional data — everything it needs arrives in the event payload. This is enforced by the event design — see the updated event catalog in `infrastructure_eventbus.md`.

---

## Events Subscribed

### `CommitmentCreatedEvent`

→ writes `thread_spun` record. If this is the user's first ever commitment → writes `first_thread` record instead.

First-ever check: reads `UserProfile.hasCompletedOnboarding` and portfolio size. If portfolio size was 0 before this creation → `first_thread`.

Narrative examples:

- `"You spun your first thread: Morning Walk"`
- `"New thread spun: No alcohol"`

---

### `CommitmentCompletedEvent`

→ writes `thread_completed` record.

Narrative: `"Morning Walk — completed. The thread holds."`

---

### `StreakMilestoneReachedEvent`

→ writes `streak_milestone` record.

Narrative: `"Morning Walk: 7-day Gold streak 🥇"`

---

### `CupEarnedEvent`

→ writes `cup_earned` record.

Narrative: `"Gold cup earned — week of Mar 10"`

---

### `LevelReachedEvent`

→ writes `level_reached` record.

Narrative uses `event.levelUpMessage` verbatim — same copy as the notification. Example: `"You reached Artisan — consistent craft, recognizable work."`

---

### `BonusThreadAwardedEvent`

→ writes `bonus_thread` record.

Narrative: `"Bonus thread awarded: comeback week +1"`

Note: this event is published by `ProgressionService` when a bonus is awarded — separate from `CupEarnedEvent`. See updated `ProgressionService` doc.

---

### `CommitmentPermanentlyDeletedEvent`

→ calls `AchievementsRepository.hideRecordsForCommitment(event.definitionId)`

Does not write a record — deletion is not an achievement. Soft-hides all records for this commitment so they no longer appear in the feed.

---

## Feed Function

### `getFeed(limit?, before?)`

Returns paginated achievement records ordered by `occurredAt` descending. Excludes soft-hidden records. Used by the achievements feed screen.

### `getFeedByCategory(category, limit?)`

Returns records for one category. Used for filtered views (e.g. "only show cups").

---

## Narrative Assembly

All narrative text is assembled inside `AchievementsService` at write time. The language follows the UI vocabulary from `language_guide.md` — weaver language, positive tone, no judgment.

Narrative templates per category:

```
first_thread:       "You spun your first thread: {commitmentName}"
thread_spun:        "New thread spun: {commitmentName}"
thread_completed:   "{commitmentName} — completed. The thread holds."
streak_milestone:   "{commitmentName}: {streakCount}-day {badgeName} streak {badge}"
cup_earned:         "{cupLevelName} cup earned — week of {weekStartFormatted}"
level_reached:      event.levelUpMessage (verbatim)
bonus_thread:       "Bonus thread awarded: {triggerDescription} +{bonusAmount}"
```

---

## Backfill on First Use

When the user first opens the achievements feed in Phase 3, `AchievementsService` checks whether any records exist. If none, it runs a one-time backfill from existing stored data:

- Reads all cup records from `RewardService.fetchAllCups()` → writes `cup_earned` records
- Reads all streak milestone records from `RewardRepository.fetchAllMilestones()` → writes `streak_milestone` records
- Reads all bonus thread records from `ProgressionRepository.fetchBonusHistory()` → writes `bonus_thread` records
- Reads level achieved dates from `ProgressionService.getProgressionSummary()` → writes `level_reached` records
- Reads all commitments from `CommitmentService.getActiveCommitments()` → writes `thread_spun` records using `createdAt`

Because streak milestones and bonus threads are now stored in their own repositories, backfill is complete and accurate — no data is lost even if Achievements is built months after the app launches.

Backfilled records are written silently — no notification, no feed refresh until complete.

---

## Rules

- Never calls any feature service in response to an event — all data comes from the event payload
- Assembles narrative text at write time — never at read time
- Positive records only — no decay, no misses, no negative entries
- Idempotent — checks for duplicate records before writing (same event type + same `occurredAt` + same metadata key)
- `CommitmentPermanentlyDeletedEvent` triggers soft-hide, never a new record

---

## Dependencies

- `EventBus` — subscribes to all achievement-producing events
- `AchievementsRepository` — writes and reads achievement records
- `UserService` — reads portfolio size for first_thread detection (one-time check only)
- `RewardService` — backfill only: reads historical cup records and milestone records
- `ProgressionService` — backfill only: reads historical level dates and bonus records
- `CommitmentService` — backfill only: reads historical commitment creation dates