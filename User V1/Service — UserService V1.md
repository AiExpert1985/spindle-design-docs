
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: User
**Phase:** 1 

**Purpose:** core business logic for the user feature. Manages profile state, tier access checks, and quota tracking. Pure Dart — no Flutter imports.

---

## Core Functions

---

### `getProfile()`

Returns the current UserProfile. Called by any feature that needs tier, quota, or preference data.

---

### `updateProfile(changes)`

Saves an updated UserProfile. Called by settings screens and migration service.

---

### `canAccessFeature(feature)`

Returns true or false based on the user's current subscription tier.

- Checks `subscriptionTier` against the feature gate table in `subscription_tiers.md`
- Called by any gated feature before rendering or executing

---

### `checkAndResetInsightQuota()`

Checks whether the free-tier micro-insight quota (3/week) has been exceeded. Resets the counter if the rolling 7-day window has passed.

- Returns: `quotaRemaining: int`
- Called by the micro-insight component before generating an insight

---

### `recordNewCommitment()`

Increments `newCommitmentsThisWeek`. Resets the counter first if the rolling 7-day window has passed.

- Called by the commitment feature when a new commitment is saved
- Used by the unlock engine to evaluate the rate limit gate

---

### `recordEncouragementSent(type)`

Updates `lastEncouragementType` and timestamp.

- Called by the day celebration component after firing
- Prevents the same story type firing two days in a row

---

## Rules

- All quota and counter resets use rolling 7-day windows — not calendar weeks
- `canAccessFeature` is the single source of truth for tier gating — no feature hardcodes tier logic
- UserService never calls other feature services — communication is one way

### `completeOnboarding()`

Sets `hasCompletedOnboarding: true` on UserProfile. Called once after the user completes the onboarding opt-in step. Never called again. Called by: onboarding component (Phase 2).