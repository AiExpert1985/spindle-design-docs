**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Rewards **Phase**: 1 (Drift) · Phase 3 (Firestore)

**Purpose:** all data access for the rewards feature. Abstract interface — nothing above this layer knows whether Drift or Firestore is active. Stores two types of records: weekly cups and streak milestone history.

---

## Weekly Cups

|Operation|Input|Output|
|---|---|---|
|`saveCup`|WeeklyCup|—|
|`cupExistsForWeek`|weekStart: DateTime|bool|
|`fetchAllCups`|—|List of WeeklyCup ordered by date desc|
|`fetchCupsSince`|from: DateTime|List of WeeklyCup since date|

---

## Streak Milestones

Permanent record of every streak milestone reached. Written by `RewardService` at the same time `StreakMilestoneReachedEvent` is published. Independent of whether Achievements feature exists — the data has value on its own (Your Record history, personal stats).

### Model: `StreakMilestoneRecord`

```
id: String                  // client-generated UUID
definitionId: String
commitmentName: String      // snapshotted at write time — survives commitment rename
streakCount: int            // e.g. 7
badge: String               // e.g. '🥇'
occurredAt: DateTime        // when the milestone was reached — from ActivityRecordedEvent
createdAt: DateTime
```

|Operation|Input|Output|
|---|---|---|
|`saveMilestone`|StreakMilestoneRecord|—|
|`fetchMilestonesForCommitment`|definitionId, limit?|List ordered by occurredAt desc|
|`fetchAllMilestones`|since?: DateTime, limit?|List ordered by occurredAt desc|

---

## Rules

- Never called directly by any feature other than `RewardService`
- `cupExistsForWeek` is the idempotency check — always called before `saveCup`
- `commitmentName` on `StreakMilestoneRecord` is snapshotted at write time — never read from the definition at display time
- Milestone records are never deleted — permanent historical record
- All IDs are client-generated UUIDs — consistent across Drift and Firestore
- No business logic — only fetch and store
- Firestore implementation scopes all queries to `/users/{userId}/` subcollections

---

## Firestore Paths

```
/users/{userId}/weeklyCups/{id}
/users/{userId}/streakMilestones/{id}
```

Both covered by the existing security rule. No additional configuration needed.