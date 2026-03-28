**File Name**: service_user_capability **Feature**: UserSettings **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains what the user is currently allowed to do. Calculates and exposes a `canAddCommitment` signal that the UI observes. The add commitment button reads this signal — it never evaluates access itself.

---

## Why This Lives in UserSettings, Not Commitment

Access rules are a property of the user — "what is this user currently allowed to do?" They are not a property of the commitment feature. As the app grows, more capability checks may appear (unlock AI Insights, export data, advanced features). All of them belong here, in one place, owned by the feature that represents the user.

Putting this logic inside CommitmentService would require CommitmentService to call PerformanceService and UserSettingsService — features above it in the dependency chain. That violates the one-way dependency rule. UserSettings sits at the top of the chain and can read from Commitment and Performance without any violation.

This is not a security-critical system. The consequence of bypassing the limit is the user creates more commitments than their tier allows — not a risk. UI-level enforcement backed by this service is appropriate for a local-first mobile app.

---

## Access Conditions for Adding a Commitment

All three must pass.

**Tier ceiling** — free tier: active commitment count must be below `AppConfig.freePortfolioLimit`. Pro and Premium: no ceiling.

**Rate limit** — fewer than `AppConfig.commitmentRateLimitMax` new commitments created within the last `AppConfig.commitmentRateLimitWindowDays` days. Prevents creating many commitments in a short burst.

**Performance threshold** — average performance over the last `AppConfig.commitmentAccessPerformanceWindowDays` days must be at or above `AppConfig.commitmentAccessPerformanceThreshold`. Encourages maintaining existing commitments before adding more.

---

## Onboarding Bypass

The first `AppConfig.onboardingBypassCount` commitments bypass all conditions. New users must be able to set up initial commitments without being blocked by performance thresholds they have not had time to meet.

Active while `UserSettingsProfile.hasCompletedOnboarding != true` and portfolio size is below the bypass count. Null and false both treated as onboarding not completed.

---

## How the UI Uses This

The add commitment button watches `canAddCommitment` via a Riverpod provider:

- `true` → button active, tap opens the commitment form
- `false` → button dimmed, tap shows a dialog with the specific reason

```dart
enum BlockedReason { tierCeiling, rateLimitExceeded, performanceBelowThreshold }

BlockedReason? getBlockedReason()   // null if not blocked
```

The button never calls any evaluation function — it only observes the provider result.

---

## Recalculation Triggers

`canAddCommitment` is recalculated when any of its inputs may have changed:

- `CommitmentService.onInstanceCreated` — commitment count changed
- `CommitmentService.onInstanceUpdated` — commitment state changed
- `CommitmentService.onInstancePermanentlyDeleted` — portfolio size changed
- `TemporalHelperService.onDayStarted` — rate limit rolling window may have shifted
- User subscription tier change — tier ceiling may have changed

**Why `onDayStarted` instead of tick-based boundary detection:** `TemporalHelperService` already detects day boundaries and publishes `DayStartedEvent`. Subscribing here avoids duplicating boundary detection logic and is consistent with the rule that all features react to boundary events from `TemporalHelperService`.

---

## Rules

- The only service that writes `canAddCommitment`
- Recalculation triggered by events — never by the presentation layer
- All thresholds come from `AppConfig` — never hardcoded
- `getBlockedReason()` is a pure read — never triggers recalculation
- Onboarding bypass checked first — short-circuits all other conditions

---

## Dependencies

- `CommitmentService` — subscribes to `onInstanceCreated`, `onInstanceUpdated`, `onInstancePermanentlyDeleted`; calls `getPortfolioSize()`, `getActiveCount()`, `getRecentlyCreated()`
- `TemporalHelperService` — subscribes to `onDayStarted`
- `PerformanceService.getPerformanceForPeriod()` — access window performance
- `UserCoreService.getTier()` — subscription tier
- `UserSettingsService` — reads onboarding status
- `AppConfig` — all thresholds and limits