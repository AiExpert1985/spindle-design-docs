**File Name**: feature_usersettings **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

UserSettings is the top of the dependency chain. It is the only writer of both user profiles, the only place that determines what the user is currently allowed to do, and the owner of all behavioral state that does not belong to any specific commitment or week. Everything in the app eventually reads from below here — subscription tier, temporal preferences, notification settings, quota, onboarding status.

---

## Why It Exists

Two reasons. First, all preference writes need a single owner — having any feature write to user profiles directly would scatter write responsibility and create races. `UserSettingsService` is the one writer of `UserCoreProfile` and `UserSettingsProfile`. Every preference change goes through it. Second, capability decisions — whether the user can add a commitment right now — require data from multiple features below (commitment count, recent performance, subscription tier). Only a feature at the top of the chain can read all of them without violating the one-way dependency rule.

---

## Position in the System

Sits at the top of the dependency chain — above Commitment, Performance, Activity, Achievements, Progression, and all features below. Reads from all of them for capability evaluation. Nothing above it in the chain — the Settings screen and presentation layer interact with it directly.

`UserCoreService` (read-only) and `UserSettingsService` (read/write) are two separate services for the same feature. Features that only need to read preferences call `UserCoreService` directly and do not depend on `UserSettingsService`. This keeps lower features' dependency footprint small.

---

## How It Works

### Two Profiles, Clear Separation

**`UserCoreProfile`** — the foundational record. Identity, subscription tier, storage backend, temporal preferences, activity window defaults, notification toggles. Read by many features throughout the app.

**`UserSettingsProfile`** — high-level behavioral state. Encouragement tracking, AI quota, onboarding status, referral. Read by fewer features, written less frequently.

The split exists because `UserCoreProfile` is read constantly — temporal preferences power every boundary calculation, activity window defaults power every new commitment. `UserSettingsProfile` holds state that is only relevant to specific features. Keeping them separate means most features only depend on `UserCoreProfile` via `UserCoreService`.

### Capability Gating — `UserCapabilityService`

`UserCapabilityService` is internal to this feature. It answers one question: can the user add a commitment right now? Three conditions must all pass — tier ceiling (free tier has a portfolio limit), rate limit (max new commitments in a rolling window), and performance threshold (average performance over a recent window must meet a minimum). New users bypass all three conditions for the first few commitments — they cannot be blocked by a performance threshold they have not had time to meet.

The result is exposed as a live Riverpod provider. The add-commitment button observes it — never evaluates the conditions itself. If blocked, `getBlockedReason()` returns which condition failed.

**Why capability lives here, not in Commitment.** Access rules are a user property, not a commitment property. If they lived in `CommitmentService`, that service would need to call `PerformanceService` and `UserSettingsService` — both above it in the chain, which is a dependency violation. UserSettings sits at the top and can read freely from both.

### Encouragement Tracking

`EncouragementService` writes one field here after emitting a day celebration: `lastEncouragementType`. This prevents the same story type from firing on consecutive days. The value is cleared after two days to prevent stale deduplication from permanently suppressing any story type.

### AI Quota

Free-tier users have a configurable weekly limit on micro-insight generation. `checkAndDecrementInsightQuota()` manages the rolling 7-day window, checks remaining quota, decrements, and returns the remaining count. Called only by `AIInsightService` — never directly by the presentation layer.

---

## Rules

- The only writer of `UserCoreProfile` and `UserSettingsProfile` — except `MigrationService` which writes `storageBackend`
- Features that only need preference reads call `UserCoreService` directly — they do not depend on `UserSettingsService`
- All temporal preferences are read through `TemporalHelperService` — never directly from `UserCoreProfile`
- Capability conditions and thresholds all come from `AppConfig` — never hardcoded
- `checkAndDecrementInsightQuota()` called only by `AIInsightService`
- `canAddCommitment` is not stored — computed live and exposed via Riverpod provider
- `activatedReferralCount` updated server-side only — never written by the app

---

## Later Improvements

**Referral system (Phase 3, Pro/Premium).** Referral code generated at Pro/Premium account creation. `activatedReferralCount` incremented server-side when a referred user earns their first cup.

**Weekly report settings (Pro/Premium).** Report time and enabled toggle — already on `UserSettingsProfile`, Settings screen UI is Phase 2.

**Onboarding flow (Phase 2).** `hasCompletedOnboarding` is already on `UserSettingsProfile` — null in Phase 1, never checked. Phase 2 adds the onboarding screen and sets this flag.


---

## Related Docs

[[Model — User Settings Profile]]
[[P2 Component — Onboarding]]
[[P3 Component — Subscription Tiers]]
[[Repository — User Settings]]
[[Screen — Recycle Bin]]
[[Screen — Referral]]
[[Screen — Settings]]
[[Service - User Capability]]
[[Service — Migration]]
[[Service — User Settings]]

