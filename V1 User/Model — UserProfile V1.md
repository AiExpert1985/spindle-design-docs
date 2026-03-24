**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: User **Phase**: 1

**Purpose:** stores all user-level preferences and state. One record per user, created on first app launch and only ever updated — never recreated.

---

## Fields

```
// Subscription
subscriptionTier: free | pro | premium
subscriptionUntil: DateTime?

// Storage
storageBackend: local | firebase

// Temporal preferences — used by TemporalHelper for all time-based conditions
weekStartDay: int            // 0=Sunday, 1=Monday, 5=Friday, 6=Saturday — default 1
restDays: List<int>          // days treated as rest days — default [6, 7] (Sat + Sun)
dayBoundaryHour: int         // hour the "day" resets (0–23) — default 0 (midnight)
wakingHoursStart: int        // default 7
wakingHoursEnd: int          // default 22

// Commitment access — maintained exclusively by CommitmentAccessService
// The add button watches this field via Riverpod — never calculates access itself
canAddCommitment: bool       // default true
                             // CommitmentAccessService sets false when any condition fails
                             // CommitmentAccessService is the ONLY writer of this field

// Day celebration settings
celebrationEnabled: bool             // default true
celebrationTime: TimeOfDay           // default 9pm
celebrationThreshold: int            // default 60
lastEncouragementType: String?       // prevents same story type two days in a row

// Weekly report settings (Pro/Premium only)
weeklyReportEnabled: bool            // default true
weeklyReportTime: TimeOfDay          // default 9pm

// AI quota (free tier only)
microInsightsUsedThisWeek: int
insightsResetDate: DateTime

// Unlock engine counters (read by CommitmentAccessService)
newCommitmentsThisWeek: int
newCommitmentsWindowStart: DateTime

// Referral (Phase 3 — Pro/Premium only)
referralCode: String?                // generated once at account creation
activatedReferralCount: int          // updated server-side via Firebase Cloud Function

updatedAt: DateTime
```

---

## Temporal Preference Defaults

Defaults written at profile creation from `UserDefaultPreferences`. Regional defaults set via onboarding one-step prompt — see `Config___UserDefaultPreferences.md`.

|Preference|Global default|Middle East|
|---|---|---|
|`weekStartDay`|1 (Monday)|6 (Saturday)|
|`restDays`|[6, 7]|[5, 6]|
|`dayBoundaryHour`|0|0|
|`wakingHoursStart`|7|7|
|`wakingHoursEnd`|22|22|

---

## Rules

- One record per user — created once on first launch, never deleted
- `storageBackend` changed only during tier migration — never manually
- `insightsResetDate` and `newCommitmentsWindowStart` use rolling 7-day windows
- `lastEncouragementType` cleared after 2 days
- `canAddCommitment` written only by `CommitmentAccessService` — no other service or UI writes it directly
- `referralCode` generated once at Pro/Premium account creation — null for free users
- `activatedReferralCount` updated server-side — never written directly by the app
- Temporal preferences read exclusively through `TemporalHelper` — no service reads them directly

---

## Fields Added by Components

```
// Onboarding (Phase 2)
hasCompletedOnboarding: bool?
```