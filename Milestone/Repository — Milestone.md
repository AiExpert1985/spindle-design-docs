**File Name**: repository_milestone **Feature**: Milestones **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage for `MilestoneRecord`. Called only by `MilestoneService`.

---

## Interface

```dart
abstract class MilestoneRepository {
  Future<void> saveMilestone(MilestoneRecord record);
  Future<bool> milestoneExists(String definitionId, int streakCount);
  Future<List<MilestoneRecord>> getMilestonesForCommitment(String definitionId, DateTime from, DateTime to);
  Future<List<MilestoneRecord>> getMilestonesForPeriod(DateTime from, DateTime to);
  Future<void> deleteMilestonesForCommitment(String definitionId);
}
```

`saveMilestone` — appends a new record. Never called for updates.

`milestoneExists` — idempotency check. Always called before `saveMilestone`.

`getMilestonesForCommitment` — all milestone records for one commitment within a date range, ordered by `createdAt` descending. Uses an equality filter on `definitionId` plus a range filter on `createdAt` — compatible with Firestore's one-range-filter constraint.

`getMilestonesForPeriod` — all milestone records across all commitments within a date range, ordered by `createdAt` descending. Used by `AchievementService` for unified achievement history. Range filter on `createdAt` only — compatible since the query is scoped to the user's subcollection.

Milestones are inherently rare events — one per streak milestone threshold crossed. The collection is naturally bounded. The `from/to` window is still required to comply with the database architecture rule and to keep queries predictable as history grows.

`deleteMilestonesForCommitment` — deletes all records for one commitment. Called only on `InstancePermanentlyDeletedEvent` cascade.

---

## Firestore Path

```
/users/{userId}/milestones/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `MilestoneService`
- Append-only except cascade delete on commitment permanent deletion
- `milestoneExists` always called before `saveMilestone`
- `deleteMilestonesForCommitment` called only on `InstancePermanentlyDeletedEvent`
- All read queries use a time window — never fetch unbounded collections