**File Name**: service_user **Feature**: User **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** manages user profile state and all user preferences. The single writer of `UserProfile`. Thin logic — reads and writes preferences, manages quotas, handles onboarding state. `SettingsService` is not a separate service — all preference functions live here.

---

## Profile Functions

### `getProfile()` → UserProfile

Returns the current `UserProfile`. Creates it with defaults from `UserDefaultPreferences` if it doesn't exist yet. Called by any feature that needs tier, quota, or preference data.

### `getTier()` → SubscriptionTier

Returns current subscription tier. Called by any feature that gates on tier.

### `getPreferences()` → UserProfile

Returns full profile. Called when the Settings screen opens and by services that read preference fields.

### `updateProfile(changes)` → void

Saves an updated `UserProfile`. Called by the Settings screen and `MigrationService`.

---

## Temporal Preference Functions

### `getTemporalPreferences()` → TemporalPreferences

Returns `weekStartDay`, `restDays`, `dayBoundaryHour`, `wakingHoursStart`, `wakingHoursEnd`. Called by `TemporalHelper` on every time-based calculation.

### `updateWeekStartDay(day)` → void

### `updateRestDays(days)` → void

### `updateDayBoundaryHour(hour)` → void

### `updateWakingHours(start, end)` → void

All temporal changes take effect immediately — `TemporalHelper` reads live from `UserProfile`.

---

## Notification Preference Functions

### `updateCelebrationSettings(enabled, time, threshold)` → void

Updates day celebration settings.

### `updateWeeklyReportSettings(enabled, time)` → void

Updates weekly report settings. Pro/Premium only — enforced by the Settings screen before calling.

### `updateWarningNotifications(enabled)` → void

Toggles warning notifications globally.

### `updateGraceSettings(enabled, minutes)` → void

Updates grace period settings.

---

## Encouragement Functions

### `recordEncouragementSent(type)` → void

Updates `lastEncouragementType` and clears it after 2 days. Called by `EncouragementService` after emitting `DayCelebrationSignal`.

---

## Quota Functions

### `checkAndDecrementInsightQuota()` → int

Checks free-tier micro-insight quota. Resets the counter if the rolling 7-day window has passed. Returns remaining quota. Called by the micro-insight component before generating.

---

## Onboarding Functions

### `completeOnboarding()` → void

Sets `hasCompletedOnboarding: true`. Called once after the onboarding opt-in step. Never called again.

---

## Rules

- The only writer of `UserProfile` — except `MigrationService` which writes `storageBackend`
- `canAddCommitment` is not stored on `UserProfile` — computed live by `UserCapabilityService`
- `newCommitmentsThisWeek` is not stored on `UserProfile` — `UserCapabilityService` reads from `CommitmentService.getRecentlyCreated()` directly
- All temporal preferences are read through `TemporalHelper` — no feature reads them directly from this service
- `SettingsService` does not exist as a separate service — all preference functions live here

---

## Dependencies

- `UserRepository` — reads and writes `UserProfile`
- `UserDefaultPreferences` — default values at profile creation