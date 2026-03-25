**File Name**: repository_progression **Feature**: Progression **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage interface for the Progression feature. Stores `ProgressionProfile` only. Called only by `ProgressionService`.

---

## ProgressionProfile Operations

|Operation|Input|Output|
|---|---|---|
|`getProfile`|—|ProgressionProfile?|
|`saveProfile`|ProgressionProfile|— full replace|
|`watchProfile`|—|Stream<ProgressionProfile?>|

`getProfile` returns null if no profile exists yet — first `PointsAwardedEvent` triggers creation.

---

## Firestore Path

```
/users/{userId}/progression/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `ProgressionService`
- No business logic — only fetch and store
- `saveProfile` is always a full replace — no partial updates
- `watchProfile` used by the Progression screen for real-time updates