**File Name**: repository_usersettings **Feature**: UserSettings **Phase**: 1 (Drift) · Phase 3 (Firestore) **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** storage interface for `UserSettingsProfile`. The high-level user state record — encouragement tracking, AI quota, onboarding, referral. Called only by `UserSettingsService`.

`UserCoreProfile` (preferences, tier, temporal settings) has its own repository inside `feature_usercore`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`getSettingsProfile`|—|UserSettingsProfile?|
|`saveSettingsProfile`|UserSettingsProfile|— full replace|
|`watchSettingsProfile`|—|Stream<UserSettingsProfile?>|

`getSettingsProfile` returns null on first launch — `UserSettingsService` creates the profile with defaults from `UserDefaultPreferences`.

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
- `saveSettingsProfile` is always a full replace — no partial updates