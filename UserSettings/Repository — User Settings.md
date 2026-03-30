**File Name**: repository_usersettings **Feature**: UserSettings **Phase**: 1 (Drift) · Phase 3 (Firestore) **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** storage interface for `UserSettingsProfile`. Called only by `UserSettingsService`.

`UserCoreProfile` (preferences, tier, temporal settings) has its own repository inside the UserCore feature.

---

## Interface

```dart
abstract class UserSettingsRepository {
  Future<UserSettingsProfile?> getSettingsProfile();
  Future<void> saveSettingsProfile(UserSettingsProfile profile);
  Stream<UserSettingsProfile?> watchSettingsProfile();
}
```

`getSettingsProfile` — returns null on first launch. `UserSettingsService` creates the profile with defaults from `UserDefaultPreferences` when null is returned.

`saveSettingsProfile` — full replace. No partial updates.

`watchSettingsProfile` — live stream. Used by screens that observe preference state.

---

## Firestore Path

```
/users/{userId}/settingsProfile/{id}    ← single document
```

Covered by existing security rule.

---

## Rules

- Called only by `UserSettingsService`
- One record only — never create a second profile
- `saveSettingsProfile` is always a full replace