**File Name**: model_usersettings_profile **Feature**: UserSettings **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the high-level user state that depends on app behavior — encouragement tracking, AI quota, onboarding state, and referral. One record per user. Low-level preferences (temporal settings, activity window defaults, tier, grace, notification toggles) live in `UserCoreProfile` — see `feature_usercore`.

---

## Fields

```
UserSettingsProfile
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

Temporal preferences (weekStartDay, restDays, dayBoundaryHour, wakingHoursStart, wakingHoursEnd) live in `UserCoreProfile`. Written once at profile creation from `UserDefaultPreferences` via `UserCoreService`.

---

## Rules

- One record per user — created on first launch, never deleted
- `lastEncouragementType` cleared after 2 days — `UserSettingsService` handles this
- `referralCode` generated once at Pro/Premium creation — null for free users
- `activatedReferralCount` updated server-side — never written by the app directly
- Written only by `UserSettingsService`