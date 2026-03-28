**File Name**: service_usersettings **Feature**: UserSettings **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the only writer of both `UserCoreProfile` and `UserSettingsProfile`. Manages high-level user state — encouragement tracking, AI quota, onboarding, subscription, and referral. Also owns all preference write functions. Sits at the top of the feature chain because `UserCapabilityService` (internal to this feature) depends on `CommitmentService` and `PerformanceService`.

Low-level preference reads are served by `UserCoreService` — features that only need to read preferences call `UserCoreService` directly without depending on `UserSettingsService`.

---

## Profile Write Functions

### `updateProfile(changes)` → void

Saves changes to `UserCoreProfile` or `UserSettingsProfile` depending on which fields changed. Called by the Settings screen.

---

## Temporal Preference Functions

### `updateWeekStartDay(day)` → void

### `updateRestDays(days)` → void

### `updateDayBoundaryHour(hour)` → void

### `updateWakingHours(start, end)` → void

Write to `UserCoreProfile`. Changes take effect immediately — `TemporalHelperService` caches preferences and refreshes via `UserCoreService.watchProfile()`.

---

## Notification Preference Functions

### `updateCelebrationSettings(enabled, time, threshold)` → void

Updates day celebration settings on `UserSettingsProfile`.

### `updateWeeklyReportSettings(enabled, time)` → void

Updates weekly report settings on `UserSettingsProfile`. Pro/Premium only — enforced by the Settings screen before calling.

### `updateWarningNotifications(enabled)` → void

Writes `warningNotificationsEnabled` to `UserCoreProfile`.

---

## Encouragement Functions

### `recordEncouragementSent(type)` → void

Updates `lastEncouragementType` on `UserSettingsProfile`. Clears it (sets to null) if the last encouragement was sent more than 2 days ago — prevents stale deduplication from affecting story selection indefinitely. Called by `EncouragementService` after emitting `DayCelebrationSignal`.

### `getLastEncouragementType()` → int?

Returns `lastEncouragementType` from `UserSettingsProfile`. Returns null if no story has been sent yet or if the value was cleared. Called by `EncouragementService` for story deduplication.

---

## Quota Functions

### `checkAndDecrementInsightQuota()` → Result<int>

Manages the free-tier micro-insight quota on `UserSettingsProfile`. Called only by `AIInsightService` — never by the presentation layer directly.

```
profile = getSettingsProfile()

// start rolling window on first call
if profile.insightsResetDate == null:
  profile.insightsResetDate = now
  profile.microInsightsUsedThisWeek = 0

// reset if 7-day window has passed
if now > profile.insightsResetDate + 7 days:
  profile.insightsResetDate = now
  profile.microInsightsUsedThisWeek = 0

// check quota
if profile.microInsightsUsedThisWeek >= AppConfig.freeInsightsPerWeek:
  return Failure(AppError(type: quota, message: 'Weekly insight quota exhausted'))

// decrement
profile.microInsightsUsedThisWeek += 1
saveSettingsProfile(profile)
return Success(AppConfig.freeInsightsPerWeek - profile.microInsightsUsedThisWeek)
```

Returns remaining quota on success, or a `Failure` with `type: quota` when exhausted. The presentation layer receives this failure from `AIInsightService` and shows the upgrade prompt.

---

## Onboarding Functions

### `completeOnboarding()` → void

Sets `hasCompletedOnboarding: true` on `UserSettingsProfile`. Called once after the onboarding opt-in step completes.

---

## Rules

- The only writer of `UserCoreProfile` and `UserSettingsProfile` — except `MigrationService` which writes `storageBackend` on `UserCoreProfile`
- `canAddCommitment` is not stored — computed live by `UserCapabilityService`
- Features that only need preference reads call `UserCoreService` directly — they do not depend on `UserSettingsService`
- `checkAndDecrementInsightQuota()` called only by `AIInsightService` — never by the presentation layer
- All temporal preferences read by other features exclusively through `TemporalHelperService`

---

## Dependencies

- `UserCoreRepository` — writes `UserCoreProfile`
- `UserSettingsRepository` — reads and writes `UserSettingsProfile`
- `UserDefaultPreferences` — default values at profile creation
- `AppConfig` — `freeInsightsPerWeek`