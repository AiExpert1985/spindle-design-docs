**File Name**: repository_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** storage interface for the Garment feature. Stores `GarmentProfile` and `GarmentDayRecord` records. Called only by `GarmentService`.

---

## GarmentProfile Operations

|Operation|Input|Output|
|---|---|---|
|`saveProfile(profile)`|GarmentProfile|— upsert|
|`getProfile(definitionId)`|definitionId|GarmentProfile?|
|`watchProfile(definitionId)`|definitionId|Stream<GarmentProfile?>|
|`deleteProfile(definitionId)`|definitionId|—|

---

## GarmentDayRecord Operations

|Operation|Input|Output|
|---|---|---|
|`saveDayRecord(record)`|GarmentDayRecord|— upsert|
|`getDayRecord(definitionId, date)`|definitionId, date|GarmentDayRecord?|
|`getDayRecords(definitionId, from, to)`|definitionId, date range|List<GarmentDayRecord>|
|`deleteDayRecords(definitionId)`|definitionId|— deletes all records for commitment|

`getDayRecord` returns null if no record exists for that date — callers must handle null defensively.

`getDayRecords` uses a single range filter on `date` combined with an equality filter on `definitionId` — compatible with Firestore's one-range-filter constraint.

---

## Firestore Paths

```
/users/{userId}/garmentProfiles/{definitionId}
/users/{userId}/garmentDayRecords/{id}
```

Both covered by the existing security rule.

---

## Rules

- Called only by `GarmentService` — no other feature accesses this repository
- No business logic — only fetch and store
- All IDs are client-generated
- `deleteProfile` and `deleteDayRecords` called only on `InstancePermanentlyDeletedEvent` cascade
- `watchProfile` stream detaches when the screen observing it is no longer visible