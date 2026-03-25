**File Name**: service_milestone **Feature**: Achievements (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** detects when a streak crosses a milestone threshold and writes a permanent record. Internal to Achievements. Subscribes to the internal `StreakChangedEvent` published by `StreakService`.

---

## Events Subscribed

### `StreakChangedEvent` (internal) → `_onStreakChanged(event)`

Only processes positive streaks — milestones are achievements, not punishments. Negative streaks are ignored.

```
if event.currentStreak <= 0: return

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
publish MilestoneReachedInternalEvent(record)
```

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Deletes all `MilestoneRecord` entries for this `definitionId`.

---

## Events Published (internal)

```
MilestoneReachedInternalEvent
  record: MilestoneRecord
```

Consumed only by `AchievementService` within the Achievements feature.

---

## Rules

- Internal to Achievements — never called directly by features outside
- Only fires for positive streak milestones — negative streaks never trigger milestones
- Idempotent — one record per commitment per milestone value
- `commitmentName` snapshotted at write time — record stays readable after rename or deletion

---

## Dependencies

- EventBus — subscribes to `StreakChangedEvent` (internal), `InstancePermanentlyDeletedEvent`; publishes `MilestoneReachedInternalEvent`
- `MilestoneRepository` — reads and writes `MilestoneRecord`
- `CommitmentService.getDefinition()` — reads commitment name for snapshot
- `AppConfig.streakMilestones` — milestone threshold values