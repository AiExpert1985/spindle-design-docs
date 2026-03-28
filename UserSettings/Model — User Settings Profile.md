**File Name**: model_usersettings_profile **Feature**: UserSettings **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the high-level user behavioral state — encouragement tracking, AI quota, onboarding state, and referral. One record per user. Low-level preferences (temporal settings, tier, notification toggles) live in `UserCoreProfile`.

---

## Fields

```dart
class UserSettingsProfile {
  final String id;

  // Encouragement preferences
  final bool celebrationEnabled;       // default true
  final int celebrationTime;           // minutes from midnight — default 1260 (21:00)
  final int celebrationThreshold;      // minimum day score % — default 60
  final int? lastEncouragementType;    // prevents same story type two consecutive days
                                       // null means no story sent yet

  // Weekly report preferences (Pro/Premium only)
  final bool weeklyReportEnabled;      // default true
  final int weeklyReportTime;          // minutes from midnight — default 1260 (21:00)

  // AI quota (free tier only)
  final int microInsightsUsedThisWeek; // default 0
  final DateTime? insightsResetDate;   // rolling 7-day window start
                                       // null on first launch — means no insights used yet

  // Onboarding (Phase 2)
  final bool? hasCompletedOnboarding;  // null in Phase 1 — never checked

  // Referral (Phase 3 — Pro/Premium only)
  final String? referralCode;          // generated once at Pro/Premium account creation
  final int activatedReferralCount;    // updated server-side — never written by app directly

  final DateTime createdAt;
  final DateTime updatedAt;
}
```

---

## Field Notes

**`lastEncouragementType`** — null means no story has been sent yet. Cleared after 2 days by `UserSettingsService` to prevent stale deduplication.

**`insightsResetDate`** — null on first launch means the rolling 7-day window has not started. `UserSettingsService.checkAndDecrementInsightQuota()` starts the window on the first insight request. When `now > insightsResetDate + 7 days`, the counter resets and a new window begins.

**`hasCompletedOnboarding`** — null in Phase 1, never checked. Phase 2 adds the onboarding flow. Null and false both mean onboarding not completed — the onboarding bypass in `UserCapabilityService` treats both identically.

**`activatedReferralCount`** — incremented server-side via Firebase Cloud Function when a referred user earns their first cup. The app reads this value but never writes it.

---

## Rules

- One record per user — created on first launch, never deleted
- `lastEncouragementType` cleared after 2 days — handled by `UserSettingsService`
- `referralCode` generated once at Pro/Premium account creation — null for free users
- `activatedReferralCount` updated server-side only — never written by the app
- Written only by `UserSettingsService`
- `createdAt` immutable — set once at profile creation