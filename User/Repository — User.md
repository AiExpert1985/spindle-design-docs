**File Name**: repository_user **Feature**: User **Phase**: 1 (Drift) · Phase 3 (Firestore) **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** all data access for the User feature. Manages the single `UserProfile` record. Abstract interface — nothing above this layer knows whether Drift or Firestore is active. Called only by `UserService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`getProfile`|—|UserProfile?|
|`saveProfile`|UserProfile|— full replace|
|`watchProfile`|—|Stream<UserProfile?>|

`getProfile` returns null on first launch — `UserService` creates the profile with defaults from `UserDefaultPreferences`.

---

## Firestore Path

```
/users/{userId}/profile/{id}    ← single document
```

Covered by existing security rule.

---

## Rules

- Called only by `UserService`
- One record only — never create a second profile
- `saveProfile` is always a full replace — no partial updates
- `watchProfile` used by any screen that needs live preference updates