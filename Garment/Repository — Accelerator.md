**File Name**: repository_accelerator **Feature**: Garment **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage interface for `AcceleratorRecord`. One record per commitment. Called only by `AcceleratorService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveRecord(record)`|AcceleratorRecord|— upsert|
|`getRecord(definitionId)`|definitionId|AcceleratorRecord?|
|`deleteRecord(definitionId)`|definitionId|—|

`getRecord` returns null if no record exists yet — `AcceleratorService` creates it on the first `StreakChangedEvent` for this commitment, initialising `multiplierValue` at 1.0.

No stream needed — `GarmentService` reads the multiplier synchronously via `AcceleratorService.getMultiplier()` at calculation time.

---

## Firestore Path

```
/users/{userId}/acceleratorRecords/{definitionId}
```

`definitionId` is the document ID. One document per commitment. Covered by the existing security rule.

---

## Rules

- Called only by `AcceleratorService`
- No business logic — only fetch and store
- `deleteRecord` called only on `InstancePermanentlyDeletedEvent` cascade