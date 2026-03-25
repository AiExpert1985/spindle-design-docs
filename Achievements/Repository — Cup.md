**File Name**: repository_cup **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage for `WeeklyCup` records. Called only by `CupService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveCup(cup)`|WeeklyCup|—|
|`cupExistsForWeek(weekStart)`|weekStart|bool — idempotency check|
|`getAllCups()`|—|List ordered by weekStart desc|
|`getCupsSince(from)`|from: DateTime|List since date|

---

## Firestore Path

```
/users/{userId}/weeklyCups/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `CupService`
- Append-only — never edited after creation
- `cupExistsForWeek` always called before `saveCup`