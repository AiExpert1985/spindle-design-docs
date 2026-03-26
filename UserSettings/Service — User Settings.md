**File Name**: service_usersettings **Feature**: UserSettings **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the only writer of both `UserCoreProfile` and `UserSettingsProfile`. Manages high-level user state — encouragement tracking, AI quota, onboarding, subscription, and referral. Also owns all preference write functions. Sits at the top of the feature chain because `UserCapabilityService` (internal to this feature) depends on `CommitmentService` and `PerformanceService`.

Low-level preference reads are served by `UserCoreService` — see `feature_usercore`. Features that only need to read preferences call `UserCoreService` directly without depending on `UserSettingsService`.

---

## Profile Write Functions

### `updateProfile(changes)` → void

Saves changes to `UserCoreProfile` or `UserSettingsProfile` depending on which fields changed. Called by the Settings screen and `MigrationService`.

---

## Temporal Preference Functions

### `updateWeekStartDay(day)` → void

### `updateRestDays(days)` → void

### `updateDayBoundaryHour(hour)` → void

### `updateWakingHours(start, end)` → void

Write to `UserCoreProfile`. Changes take effect immediately — `TemporalHelper` reads live from `UserCoreProfile` via `UserCoreService`.

---

## Notification Preference Functions

### `updateCelebrationSettings(enabled, time, threshold)` → void

Updates day celebration settings on `UserSettingsProfile`.

### `updateWeeklyReportSettings(enabled, time)` → void

Updates weekly report settings. Pro/Premium only — enforced by the Settings screen before calling.

### `updateWarningNotifications(enabled)` → void

Writes `warningNotificationsEnabled` to `UserCoreProfile`.

### `updateGraceSettings(enabled, minutes)` → void

Writes grace preferences to `UserCoreProfile`.

---

## Encouragement Functions

### `recordEncouragementSent(type)` → void

Updates `lastEncouragementType` on `UserSettingsProfile` and clears it after 2 days. Called by `EncouragementService` after emitting `DayCelebrationSignal`.

---

## Quota Functions

### `checkAndDecrementInsightQuota()` → int

Checks free-tier micro-insight quota on `UserSettingsProfile`. Resets the counter if the rolling 7-day window has passed. Returns remaining quota. Called by the micro-insight component before generating.

---

## Onboarding Functions

### `completeOnboarding()` → void

Sets `hasCompletedOnboarding: true` on `UserSettingsProfile`. Called once after the onboarding opt-in step.

---

## Rules

- The only writer of `UserCoreProfile` and `UserSettingsProfile` — except `MigrationService` which writes `storageBackend` on `UserCoreProfile`
- `canAddCommitment` is not stored — computed live by `UserCapabilityService`
- Features that only need preference reads call `UserCoreService` directly — they do not depend on `UserSettingsService`
- All temporal preferences read by other features exclusively through `TemporalHelper`

---

## Dependencies

- `UserCoreRepository` — writes `UserCoreProfile`
- `UserSettingsRepository` — writes `UserSettingsProfile`
- `UserDefaultPreferences` — default values at profile creation