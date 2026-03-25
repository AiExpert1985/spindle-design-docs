**File Name**: model_user_profile **Feature**: User **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** stores all user-level state and preferences. One record per user, created on first app launch and only ever updated — never recreated.

---

## Fields

```
UserProfile
  // Subscription (Phase 3)
  subscriptionTier: SubscriptionTier     // free | pro | premium — default free
  subscriptionUntil: DateTime?           // null for free tier

  // Storage (Phase 3)
  storageBackend: StorageBackend         // local | firebase

  // Temporal preferences
  weekStartDay: int                      // 0=Sunday 1=Monday 5=Friday 6=Saturday — default 1
  restDays: List<int>                    // default [6, 7] (Sat + Sun)
  dayBoundaryHour: int                   // 0–23 — default 0 (midnight)
  wakingHoursStart: int                  // default 7
  wakingHoursEnd: int                    // default 22

  // Activity window defaults
  defaultDailyWindowStart: int           // minutes from midnight — default same as dayBoundaryHour
  defaultDailyWindowDuration: int        // minutes — default 1440 (24 hours)
  defaultWeeklyWindowStart: int          // minutes from midnight on weekStartDay
  defaultWeeklyWindowDuration: int       // minutes — default 10080 (7 days)
  defaultSpecificDayWindowStart: int     // minutes from midnight — default same as wakingHoursStart
  defaultSpecificDayWindowDuration: int  // minutes — default 960 (16 hours)

  // Grace preferences
  graceEnabled: bool                     // default true
  gracePeriodMinutes: int                // default 15 — overrides AppConfig for this user

  // Notification preferences
  warningNotificationsEnabled: bool      // default true
  windowCloseNotificationsEnabled: bool  // default true

  // Encouragement preferences
  celebrationEnabled: bool               // default true
  celebrationTime: int                   // minutes from midnight — default 1260 (21:00)
  celebrationThreshold: int             // minimum day score % — default 60
  lastEncouragementType: int?           // prevents same story type two consecutive days
                                         // null means no story sent yet

  // Weekly report preferences (Pro/Premium only)
  weeklyReportEnabled: bool              // default true
  weeklyReportTime: int                  // minutes from midnight — default 1260 (21:00)

  // AI quota (free tier only)
  microInsightsUsedThisWeek: int         // default 0
  insightsResetDate: DateTime?           // rolling 7-day window start — null on first launch

  // Onboarding (Phase 2)
  hasCompletedOnboarding: bool?          // null in Phase 1 — never checked

  // Referral (Phase 3 — Pro/Premium only)
  referralCode: String?                  // generated once at Pro/Premium account creation
  activatedReferralCount: int            // updated server-side via Firebase Cloud Function

  updatedAt: DateTime
```

---

## Temporal Preference Defaults

Written once at profile creation from `UserDefaultPreferences`. Regional defaults applied via onboarding one-step prompt.

|Preference|Global default|Middle East|
|---|---|---|
|`weekStartDay`|1 (Monday)|6 (Saturday)|
|`restDays`|[6, 7]|[5, 6]|
|`dayBoundaryHour`|0|0|
|`wakingHoursStart`|7|7|
|`wakingHoursEnd`|22|22|

---

## Rules

- One record per user — created on first launch, never deleted
- `storageBackend` changed only by `MigrationService` — never written directly
- `lastEncouragementType` cleared after 2 days — `UserService` handles this
- `referralCode` generated once at Pro/Premium creation — null for free users
- `activatedReferralCount` updated server-side — never written by the app directly
- All temporal preferences read exclusively through `TemporalHelper` — no service reads them directly
- `microInsightsUsedThisWeek` and `insightsResetDate` use rolling 7-day windows
- Written only by `UserService` and `MigrationService`