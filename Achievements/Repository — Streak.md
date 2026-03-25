**File Name**: repository_streak **Feature**: Achievements (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage interface for the Streak feature. One `StreakRecord` per commitment. Called only by `StreakService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveRecord(record)`|StreakRecord|— upsert|
|`getRecord(definitionId)`|definitionId|StreakRecord?|
|`watchRecord(definitionId)`|definitionId|Stream<StreakRecord?>|
|`getAllRecords()`|—|List<StreakRecord>|
|`deleteRecord(definitionId)`|definitionId|—|

`getRecord` returns null if no record exists yet — `StreakService` creates it on the first window close for this commitment.

`getAllRecords` used by `StreakService` to find the global best streak across all commitments.

`watchRecord` used by the commitment detail screen for live streak display.

---

## Firestore Path

```
/users/{userId}/streakRecords/{definitionId}
```

`definitionId` is the document ID — no separate id field. One document per commitment. Covered by the existing security rule.

---

## Rules

- Called only by `StreakService`
- No business logic — only fetch and store
- `deleteRecord` called only on `InstancePermanentlyDeletedEvent` cascade
- `watchRecord` stream detached when the observing screen is no longer visible