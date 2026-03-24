**Created**: 17-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Progression **Phase**: 3 (Firestore only — Pro/Premium users are always on Firebase)

**Purpose:** all data access for the progression feature. Abstract interface — nothing above this layer knows whether Drift or Firestore is active. Stores two types of records: the progression profile (single document) and bonus thread history.

---

## Progression Profile

Single document per user.

|Operation|Input|Output|
|---|---|---|
|`getProfile`|—|ProgressionProfile?|
|`saveProfile`|ProgressionProfile|—|
|`watchProfile`|—|Stream of ProgressionProfile?|

---

## Bonus Thread History

Permanent record of every bonus thread awarded. Written by `ProgressionService` at the same time `BonusThreadAwardedEvent` is published. Independent of whether Achievements feature exists — the data has value on its own (Progression Screen history, personal stats, bonus trigger analytics).

### Model: `BonusThreadRecord`

```
id: String                  // client-generated UUID
weekStart: DateTime         // the week this bonus was awarded for
bonusAmount: double         // amount awarded (1.0 or 2.0)
trigger: String             // 'first_cup' | 'first_diamond' | 'three_consecutive'
                            // | 'comeback' | 'diamond_comeback'
occurredAt: DateTime        // when the bonus was awarded — same as processing time
createdAt: DateTime
```

|Operation|Input|Output|
|---|---|---|
|`saveBonusRecord`|BonusThreadRecord|—|
|`fetchBonusHistory`|since?: DateTime, limit?|List ordered by occurredAt desc|
|`fetchBonusSince`|from: DateTime|List — used by ProgressionService for trigger evaluation|

---

## Rules

- Never called directly by any feature other than `ProgressionService`
- `getProfile` returns null if no profile exists yet — first cup award creates the profile
- `saveProfile` is always a full replace — no partial updates
- `watchProfile` used by the Progression Screen to update in real time
- Bonus records are never deleted — permanent historical record
- All IDs follow the same client-generated UUID pattern as all other repositories
- Firestore implementation scopes all queries to `/users/{userId}/` subcollections

---

## Firestore Paths

```
/users/{userId}/progression/{id}        ← single profile document
/users/{userId}/bonusThreads/{id}       ← bonus history records
```

Both covered by the existing security rule. No additional configuration needed.