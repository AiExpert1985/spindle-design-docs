**File Name**: service_milestone **Feature**: Milestones **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** detects when a streak crosses a milestone threshold and writes a permanent `MilestoneRecord`. An independent feature — sits above Streak in the chain because it depends on `StreakChangedEvent`. Publishes `MilestoneEarnedEvent` carrying a `MilestoneRecord` which implements `Achievable`. `AchievementService` subscribes and calls `toAchievementRecord()`.

---

## Design Decisions

**Why Milestones is its own feature, not internal to Achievements.** Milestones has its own model, repository, and service — by our architecture definition, that makes it a feature. Streak, Cups, and Rewards are all independent for the same reason. Keeping Milestones internal to Achievements would make Achievements a mixed-concern feature again — the exact problem solved by separating the others. Consistency demands Milestones be independent.

**Why Milestones sits above Streak, not at the same level.** Milestones subscribes to `StreakChangedEvent` from Streak. A feature that depends on another must sit above it — same-level features never depend on each other. Milestones is therefore one level above Streak, Cups, Rewards, and Analytics.

---

## AppConfig Constants

```
streakMilestones: [7, 14, 30, 60, 90, 180, 365]
  The streak day counts that trigger a milestone. Adjustable after launch.
```

These are starting values. They represent natural behavioral checkpoints — one week, two weeks, one month, two months, three months, six months, one year. They will be tuned based on real user data after launch.

---

## Events Subscribed

### `StreakService.onStreakChanged` → `_onStreakChanged(event)`

```
if event.currentStreak <= 0: return   // only positive streaks qualify

if not AppConfig.streakMilestones.contains(event.currentStreak): return

if MilestoneRepository.milestoneExists(event.definitionId, event.currentStreak): return

definition = CommitmentService.getDefinition(event.definitionId)
commitmentName = definition?.name ?? event.definitionId  // fallback to id if name unavailable

record = MilestoneRecord(
  definitionId: event.definitionId,
  commitmentName: commitmentName,
  streakCount: event.currentStreak,
  createdAt: now,
  updatedAt: now
)
MilestoneRepository.saveMilestone(record)
publish MilestoneEarnedEvent(record: record)
```

If the definition is not found, `commitmentName` falls back to `definitionId` — the record is still written and the warning is logged via `LoggerService`. A missing definition at this point is unexpected but not fatal.

### `CommitmentService.onInstancePermanentlyDeleted` → `_onDeleted(event)`

Deletes all `MilestoneRecord` entries for this `definitionId`.

---

## Events Published

```
MilestoneEarnedEvent
  record: MilestoneRecord    // implements Achievable
```

Consumed by `AchievementService` which calls `record.toAchievementRecord()`.

---

## Read Functions

### `getMilestonesForCommitment(definitionId, from, to)` → List<MilestoneRecord>

All milestones for one commitment within a date range. Ordered by `createdAt` descending.

### `getMilestonesForPeriod(from, to)` → List<MilestoneRecord>

All milestones across all commitments within a date range. Used by `AchievementService` for unified achievement history.

---

## Rules

- Independent feature — not internal to Achievements
- Only fires for positive streak milestones — negative streaks never trigger milestones
- Idempotent — one record per commitment per milestone value
- `commitmentName` snapshotted at write time — readable after rename or deletion
- Falls back to `definitionId` if commitment name is unavailable — logs warning, never silently drops the record
- `MilestoneRecord` implements `Achievable` — translation is the model's responsibility
- All read queries use a time window — never fetch unbounded history

---

## Dependencies

- `StreakService` — subscribes to `onStreakChanged`
- `CommitmentService` — subscribes to `onInstancePermanentlyDeleted`; calls `getDefinition()` for name snapshot
- `MilestoneRepository` — reads and writes `MilestoneRecord`
- `LoggerService` — logs warning when commitment name is unavailable
- `AppConfig` — `streakMilestones`