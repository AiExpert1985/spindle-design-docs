**File Name**: service_milestone **Feature**: Milestones **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** detects when a streak crosses a milestone threshold and writes a permanent `MilestoneRecord`. An independent feature — sits above Streak in the chain because it depends on `StreakChangedEvent`. Publishes `MilestoneEarnedEvent` carrying a `MilestoneRecord` which implements `Achievable`. `AchievementService` subscribes and calls `toAchievementRecord()`.

---

## Design Decision

**Why Milestones is its own feature, not internal to Achievements.** Milestones has its own model, repository, and service — by our architecture definition, that makes it a feature. Streak, Cups, and Rewards are all independent for the same reason. Keeping Milestones internal to Achievements would make Achievements a mixed-concern feature again — the exact problem we solved by separating the others. Consistency demands Milestones be independent.

**Why Milestones sits above Streak, not at the same level.** Milestones subscribes to `StreakChangedEvent` from Streak. A feature that depends on another must sit above it — same-level features never depend on each other. Milestones is therefore one level above Streak, Cups, Rewards, and Analytics.

---

## Events Subscribed

### `StreakChangedEvent` → `_onStreakChanged(event)`

```
if event.currentStreak <= 0: return   // only positive streaks qualify

if not AppConfig.streakMilestones.contains(event.currentStreak): return

if MilestoneRepository.milestoneExists(event.definitionId, event.currentStreak): return

commitmentName = CommitmentService.getDefinition(event.definitionId)?.name ?? ''
record = MilestoneRecord(
  definitionId: event.definitionId,
  commitmentName: commitmentName,
  streakCount: event.currentStreak,
  earnedAt: now
)
MilestoneRepository.saveMilestone(record)
publish MilestoneEarnedEvent(record: record)
```

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Deletes all `MilestoneRecord` entries for this `definitionId`.

---

## Events Published

```
MilestoneEarnedEvent
  record: MilestoneRecord    // implements Achievable
```

Consumed by `AchievementService` which calls `record.toAchievementRecord()`.

---

## Rules

- Independent feature — not internal to Achievements
- Only fires for positive streak milestones — negative streaks never trigger milestones
- Idempotent — one record per commitment per milestone value
- `commitmentName` snapshotted at write time — readable after rename or deletion
- `MilestoneRecord` implements `Achievable` — translation is the model's responsibility

---

## Dependencies

- EventBus — subscribes to `StreakChangedEvent` (Streak), `InstancePermanentlyDeletedEvent` (Commitment); publishes `MilestoneEarnedEvent`
- `MilestoneRepository` — reads and writes `MilestoneRecord`
- `CommitmentService.getDefinition()` — reads commitment name for snapshot
- `AppConfig.streakMilestones`