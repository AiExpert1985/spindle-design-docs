**File Name**: repository_garment **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage interface for the Garment feature. Stores `GarmentProfile` and `CommitmentWeeklyProgress` records. Called only by `GarmentService`.

---

## GarmentProfile Operations

|Operation|Input|Output|
|---|---|---|
|`saveProfile(profile)`|GarmentProfile|— upsert|
|`getProfile(definitionId)`|definitionId|GarmentProfile?|
|`watchProfile(definitionId)`|definitionId|Stream<GarmentProfile?>|
|`deleteProfile(definitionId)`|definitionId|—|

---

## CommitmentWeeklyProgress Operations

|Operation|Input|Output|
|---|---|---|
|`saveWeeklyProgress(record)`|CommitmentWeeklyProgress|— upsert|
|`getWeeklyProgress(definitionId, limit?)`|definitionId, optional limit|List ordered by weekStart desc|
|`getLiveWeekProgress(definitionId)`|definitionId|CommitmentWeeklyProgress?|
|`deleteWeeklyProgress(definitionId)`|definitionId|— deletes all records for commitment|

---

## Firestore Paths

```
/users/{userId}/garmentProfiles/{definitionId}
/users/{userId}/weeklyProgress/{id}
```

Both covered by the existing security rule. No additional configuration needed.

---

## Rules

- Called only by `GarmentService` — no other feature accesses this repository
- No business logic — only fetch and store
- All IDs are client-generated
- `deleteProfile` and `deleteWeeklyProgress` called only on `InstancePermanentlyDeletedEvent` cascade
- `watchProfile` stream detaches when the screen observing it is no longer visible