**File Name**: repository_user_core **Feature**: UserCore **Phase**: 1 (Drift) · Phase 3 (Firestore) **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage interface for `UserCoreProfile`. Called only by `UserCoreService` for reads, and by `UserSettingsService` for writes.

---

## Interface

```dart
abstract class UserCoreRepository {
  Future<UserCoreProfile?> getProfile();
  Future<void> saveProfile(UserCoreProfile profile);
  Stream<UserCoreProfile?> watchProfile();
}
```

`getProfile()` returns null on first launch. `UserCoreService` handles null by creating the profile with defaults and writing it back through `UserSettingsService`.

`saveProfile()` is always a full replace — no partial updates.

`watchProfile()` streams live changes. Used by features that need to react when preferences change — for example, refreshing cached temporal preferences when the user updates their region settings.

---

## Firestore Path

```
/users/{userId}/coreProfile/{id}    ← single document
```

Covered by the standard security rule.

---

## Rules

- `getProfile()` and `watchProfile()` called only by `UserCoreService`
- `saveProfile()` called only by `UserSettingsService`
- One record per user — never create a second profile
- Full replace on every write — no partial field updates