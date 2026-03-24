**File Name**: repository_progression **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage interface for the Progression feature. Stores `ProgressionProfile` and `BonusThreadRecord` records. Called only by `ProgressionService`.

---

## ProgressionProfile Operations

| Operation | Input | Output |
|---|---|---|
| `getProfile` | — | ProgressionProfile? |
| `saveProfile` | ProgressionProfile | — full replace |
| `watchProfile` | — | Stream\<ProgressionProfile?\> |

`getProfile` returns null if no profile exists yet — first cup award triggers creation via `ProgressionService`.

---

## BonusThreadRecord Operations

| Operation | Input | Output |
|---|---|---|
| `saveBonusRecord` | BonusThreadRecord | — |
| `fetchBonusHistory` | since?: DateTime, limit? | List ordered by occurredAt desc |
| `fetchBonusSince` | from: DateTime | List — used for trigger evaluation |

Bonus records are never deleted — permanent historical record.

---

## Firestore Paths

```
/users/{userId}/progression/{id}       ← single profile document
/users/{userId}/bonusThreads/{id}      ← bonus history records
```

Both covered by the existing security rule. No additional configuration needed.

---

## Rules

- Called only by `ProgressionService`
- No business logic — only fetch and store
- `saveProfile` is always a full replace — no partial updates
- `watchProfile` used by the Progression screen to update in real time
- All IDs are client-generated UUIDs
