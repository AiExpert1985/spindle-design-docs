**File Name**: model_milestone_record **Feature**: Milestones **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** a permanent record of one streak milestone reached for a specific commitment. Implements `Achievable` — `AchievementService` calls `toAchievementRecord()` when `MilestoneEarnedEvent` arrives.

---

## Fields

```dart
class MilestoneRecord implements Achievable {
  final String id;
  final String definitionId;
  final String commitmentName;  // snapshotted at write time
  final int streakCount;        // the streak value that triggered this milestone
  final DateTime createdAt;
  final DateTime updatedAt;
}
```

- **commitmentName** — snapshotted at write time so the record stays readable after rename or deletion.
- **streakCount** — matches one of the values in `AppConfig.streakMilestones`.
- **createdAt** — when the milestone was earned. Immutable.
- **updatedAt** — initialized to `createdAt`. Milestones are append-only and never edited — `updatedAt` always equals `createdAt`. Both fields are present to satisfy the model convention.

---

## Achievable Implementation

```dart
@override
AchievementRecord toAchievementRecord() {
  return AchievementRecord(
    type: AchievementType.streakMilestone,
    subtype: '${streakCount}_day',    // e.g. '7_day', '14_day'
    sourceId: id,
    definitionId: definitionId,
    earnedAt: createdAt,
  );
}
```

Pure function — no side effects, no service calls.

---

## Rules

- One record per milestone per commitment — `MilestoneService` checks idempotency
- Append-only — never edited after creation
- `updatedAt` initialized to `createdAt` — milestones are never modified after creation
- Written only by `MilestoneService`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`
- `toAchievementRecord()` called only by `AchievementService`