**File Name**: service_user_capability **Feature**: UserSettings **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** maintains what the user is currently allowed to do. Calculates and exposes a `canAddCommitment` signal that the UI observes. The add commitment button reads this signal — it never evaluates access itself.

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

- `CommitmentIdentityService.onInstanceCreated` — commitment count changed
- `CommitmentIdentityService.onInstanceUpdated` — commitment state changed
- `CommitmentIdentityService.onInstancePermanentlyDeleted` — portfolio size changed
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

- `CommitmentIdentityService` — subscribes to `InstanceCreatedEvent`, `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent`; calls `getPortfolioSize()`, `getActiveCount()`, `getRecentlyCreated()`
- `TemporalHelperService` — subscribes to `DayStartedEvent`
- `PerformanceService.getPerformanceForPeriod()` — access window performance
- `UserCoreService.getTier()` — subscription tier
- `UserSettingsService` — reads onboarding status
- `AppConfig` — all thresholds and limits