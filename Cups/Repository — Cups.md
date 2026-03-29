**File Name**: repository_cup **Feature**: Cups **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** storage for `WeeklyCup` records. Called only by `CupService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveCup(cup)`|WeeklyCup|— upsert by `id`|
|`getAllCups(from, to)`|date range|List ordered by weekStart desc|

`saveCup` upserts by document ID (`year_weeknumber`). No existence check needed — writing the same week twice produces the same record.

`getAllCups` uses a single range filter on `weekStart` — compatible with Firestore's one-range-filter constraint.

---

## Firestore Path

```
/users/{userId}/weeklyCups/{year_weeknumber}
```

Document ID is the natural unique key. Covered by existing security rule.

---

## Rules

- Called only by `CupService`
- Append-only in practice — upsert of the same week produces identical data
- No existence check before write — idempotency is guaranteed by document ID