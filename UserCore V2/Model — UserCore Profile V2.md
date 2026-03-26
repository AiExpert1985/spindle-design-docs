**File Name**: model_user_core_profile **Feature**: UserCore **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the user's foundational record — identity, subscription tier, and all low-level preferences that the rest of the app reads to function. One record per user. High-level behavioral state (encouragement tracking, AI quota, onboarding, referral) lives in `UserSettingsProfile` — see `model_user_settings_profile`.

---

## Fields

```
UserCoreProfile
  // Identity
  id: String                            // stable, generated on first launch

  // Subscription (Phase 3)
  subscriptionTier: SubscriptionTier    // free | pro | premium — default free
  subscriptionUntil: DateTime?          // null for free tier

  // Storage (Phase 3)
  storageBackend: StorageBackend        // local | firebase — default local

  // Temporal preferences
  weekStartDay: int                     // 0=Sunday 1=Monday 5=Friday 6=Saturday — default 1
  restDays: List<int>                   // default [6, 7] (Sat + Sun)
  dayBoundaryHour: int                  // 0–23 — default 0 (midnight)
  wakingHoursStart: int                 // default 7
  wakingHoursEnd: int                   // default 22

  // Activity window defaults
  defaultDailyWindowStart: int          // minutes from midnight — default 0
  defaultDailyWindowDuration: int       // minutes — default 1440 (24 hours)
  defaultWeeklyWindowStart: int         // minutes from midnight on weekStartDay — default 0
  defaultWeeklyWindowDuration: int      // minutes — default 10080 (7 days)
  defaultSpecificDayWindowStart: int    // minutes from midnight — default same as wakingHoursStart
  defaultSpecificDayWindowDuration: int // minutes — default 960 (16 hours)

  // Notification preferences
  warningNotificationsEnabled: bool     // default true
  windowCloseNotificationsEnabled: bool // default true

  updatedAt: DateTime
```

---

## Default Values

Defaults are written to the profile once at first launch by `UserCoreService`. Users can change any preference through Settings. Defaults are never re-applied after creation.

### Temporal Preferences

Temporal preferences vary by region. A one-step prompt on first launch sets the correct starting point:

```
Where are you based?
[ Middle East ]   [ Europe / Americas ]   [ Other ]
```

Each selection writes the correct regional defaults. "Other" applies the global defaults.

|Preference|Global default|Middle East|
|---|---|---|
|`weekStartDay`|Monday (1)|Saturday (6)|
|`restDays`|[6, 7] Sat + Sun|[5, 6] Fri + Sat|
|`dayBoundaryHour`|0 (midnight)|0 (midnight)|
|`wakingHoursStart`|7|7|
|`wakingHoursEnd`|22|22|

### Activity Window Defaults

Applied when a new commitment is created. User can override per commitment.

|Preference|Default|
|---|---|
|`defaultDailyWindowStart`|same as `dayBoundaryHour`|
|`defaultDailyWindowDuration`|1440 min (24 hours)|
|`defaultWeeklyWindowStart`|`weekStartDay` at `dayBoundaryHour`|
|`defaultWeeklyWindowDuration`|10080 min (7 days)|
|`defaultSpecificDayWindowStart`|same as `wakingHoursStart`|
|`defaultSpecificDayWindowDuration`|960 min (16 hours)|

### Notification Preferences

|Preference|Default|
|---|---|
|`warningNotificationsEnabled`|true|
|`windowCloseNotificationsEnabled`|true|

---

## Rules

- One record per user — created once on first launch, never recreated
- `storageBackend` changed only by `MigrationService` — no other service writes this field
- Written only by `UserSettingsService` — `UserCoreService` is read-only
- System constants that are not user-configurable live in `app_configuration`