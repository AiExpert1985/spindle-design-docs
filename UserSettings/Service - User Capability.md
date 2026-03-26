**File Name**: usercapabilityservice **Feature**: UserSettings **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** maintains what the user is currently allowed to do. Calculates and exposes a `canAddCommitment` signal that the UI observes. The add commitment button reads this signal — it never evaluates access itself.

---

## Why This Lives in UserSettings, Not Commitment

Access rules are a property of the user — "what is this user currently allowed to do?" They are not a property of the commitment feature. As the app grows, more capability checks may appear (unlock AI Insights, export data, advanced features). All of them belong here, in one place, owned by the feature that represents the user.

Putting this logic inside CommitmentService would require CommitmentService to call PerformanceService and UserSettingsService — features above it in the dependency chain. That violates the one-way dependency rule. The UserSettings feature sits at the top of the chain and can read from Commitment and Performance without any violation.

This is not a security-critical system. The consequence of bypassing the limit is the user creates more commitments than their tier allows — not a risk. UI-level enforcement backed by this service is appropriate for a local-first mobile app.

---

## Access Conditions for Adding a Commitment

All three must pass.

**Tier ceiling** — free tier: active commitment count must be below `AppConfig.freePortfolioLimit`. Pro and Premium: no ceiling.

**Rate limit** — fewer than `AppConfig.commitmentRateLimitMax` new commitments created within the last `AppConfig.commitmentRateLimitWindowDays` days. Prevents creating many commitments in a short burst.

**Performance threshold** — average performance over the last `AppConfig.commitmentAccessPerformanceWindowDays` days must be at or above `AppConfig.commitmentAccessPerformanceThreshold`. Encourages the user to maintain existing commitments before adding more.

---

## Onboarding Bypass

The first `AppConfig.onboardingBypassCount` commitments bypass all conditions. New users must be able to set up initial commitments without being blocked by performance thresholds they have not had time to meet.

Active while `UserSettingsProfile.hasCompletedOnboarding == false` and portfolio size is below the bypass count.

---

## How the UI Uses This

The add commitment button watches `canAddCommitment` via a Riverpod provider:

- `true` → button active, tap opens the commitment form
- `false` → button dimmed, tap shows a dialog with the specific reason

`getBlockedReason()` returns why access is currently denied.

```
enum BlockedReason { tierCeiling, rateLimitExceeded, performanceBelowThreshold }

getBlockedReason() → BlockedReason?   // null if not blocked
```

The button never calls any evaluation function — it only observes the result.

---

## Recalculation Triggers

`canAddCommitment` is recalculated when any of its inputs may have changed:

- `InstanceCreatedEvent` or `InstanceUpdatedEvent` — commitment count or state changed
- `InstancePermanentlyDeletedEvent` — portfolio size changed
- `LongIntervalTickEvent` at day boundary — rate limit rolling window may have shifted
- User subscription tier change — tier ceiling may have changed

---

## Rules

- The only service that writes `canAddCommitment`
- Recalculation triggered by events and day boundary ticks — never by the presentation layer
- All thresholds come from AppConfig — never hardcoded
- `getBlockedReason()` is a pure read — never triggers recalculation
- Onboarding bypass checked first — short-circuits all other conditions

---

## Dependencies

- EventBus — subscribes to InstanceCreatedEvent, InstanceUpdatedEvent, InstancePermanentlyDeletedEvent, LongIntervalTickEvent
- CommitmentService — reads portfolio size and recently created count
- PerformanceService — reads average performance for the access window
- UserCoreService — reads subscription tier
- UserSettingsService — reads onboarding status
- TemporalHelper — day boundary check on tick
- AppConfig — all thresholds and limits