**File Name**: feature_usercore **Feature**: UserCore **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the lowest-level feature in the application. Owns the `UserCoreProfile` record and exposes read-only access to preferences and tier data. Every feature that needs to know about the user's configuration reads from here. `UserSettingsService` is the only writer.

---

## Why UserCore Exists as a Separate Feature

The original `User` feature mixed two concerns: low-level preference storage that everything depends on, and high-level capability logic that depends on Commitment and Performance. Keeping them together forced every feature that needed a simple preference read (like `TemporalHelper` reading `weekStartDay`) to depend on a feature that also depended on Commitment — creating a cycle or forcing an impossible chain position.

`UserCore` holds only what is structurally fundamental: the user's identity, preferences, and tier. It has no dependencies on any feature. It can be read from anywhere. `UserSettingsService` sits at the top of the chain, depends on Commitment and Performance, and is the only service that writes to `UserCore`.

---

## UserCoreProfile Model

```
UserCoreProfile
  // Identity
  id: String                           // stable, generated on first launch

  // Subscription (Phase 3)
  subscriptionTier: SubscriptionTier   // free | pro | premium — default free
  subscriptionUntil: DateTime?         // null for free tier

  // Storage (Phase 3)
  storageBackend: StorageBackend       // local | firebase — default local

  // Temporal preferences
  weekStartDay: int                    // 0=Sunday 1=Monday 5=Friday 6=Saturday — default 1
  restDays: List<int>                  // default [6, 7] (Sat + Sun)
  dayBoundaryHour: int                 // 0–23 — default 0 (midnight)
  wakingHoursStart: int                // default 7
  wakingHoursEnd: int                  // default 22

  // Activity window defaults
  defaultDailyWindowStart: int         // minutes from midnight — default 0
  defaultDailyWindowDuration: int      // minutes — default 1440 (24 hours)
  defaultWeeklyWindowStart: int        // minutes from midnight on weekStartDay
  defaultWeeklyWindowDuration: int     // minutes — default 10080 (7 days)
  defaultSpecificDayWindowStart: int   // minutes from midnight — default same as wakingHoursStart
  defaultSpecificDayWindowDuration: int // minutes — default 960 (16 hours)

  // Grace preferences
  graceEnabled: bool                   // default true
  gracePeriodMinutes: int              // default 15

  // Notification preferences
  warningNotificationsEnabled: bool    // default true
  windowCloseNotificationsEnabled: bool // default true

  updatedAt: DateTime
```

Fields that belong to higher-level concerns (encouragement tracking, AI quota, referral, onboarding state) live on `UserSettingsProfile` — see `service_usersettings`.

---

## UserCoreService

The public read interface for all features. `UserSettingsService` is the only writer — `UserCoreService` exposes no write functions.

### `getProfile() → UserCoreProfile`

Returns the current profile. Creates it with defaults from `UserDefaultPreferences` if it does not exist yet (first launch).

### `getTier() → SubscriptionTier`

Returns the current subscription tier. Called by any feature that gates behavior on tier.

### `getTemporalPreferences() → TemporalPreferences`

Returns `weekStartDay`, `restDays`, `dayBoundaryHour`, `wakingHoursStart`, `wakingHoursEnd`. Called by `TemporalHelper` on every time-based calculation.

### `getActivityWindowDefaults(recurrence) → ActivityWindowDefaults`

Returns the default `startMinutes` and `durationMinutes` for a given recurrence type. Called by `CommitmentForm` when creating a new commitment.

### `getGracePreferences() → GracePreferences`

Returns `graceEnabled` and `gracePeriodMinutes`. Called by `GraceService`.

### `watchProfile() → Stream<UserCoreProfile>`

Live stream. Used by `TemporalHelper` to refresh its cached preferences when the profile changes.

---

## UserCoreRepository

Called only by `UserCoreService` and `UserSettingsService` (for writes).

```
getProfile()        → UserCoreProfile?
saveProfile(profile) → void             // full replace — only UserSettingsService calls this
watchProfile()      → Stream<UserCoreProfile?>
```

`getProfile()` returns null on first launch. `UserCoreService.getProfile()` creates the profile with defaults when null is returned.

**Firestore path:**

```
/users/{userId}/profile/{id}    ← single document
```

---

## Rules

- `UserCoreService` exposes read functions only — no write functions
- `UserSettingsService` is the only writer of `UserCoreProfile` — calls `UserCoreRepository.saveProfile()` directly
- `storageBackend` changed only by `MigrationService` — never written by any other service
- All temporal preferences read exclusively through `TemporalHelper` — no feature reads them from `UserCoreService` directly except `TemporalHelper` itself
- Profile created once on first launch with defaults from `UserDefaultPreferences` — never recreated
- No feature events published — profile changes propagate via `watchProfile()` stream

---

## Dependencies

- `UserCoreRepository` — reads UserCoreProfile
- `UserDefaultPreferences` — default values at profile creation