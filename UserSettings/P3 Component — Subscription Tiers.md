**File Name**: component_subscription_tiers **Feature**: UserSettings **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** defines what each tier includes and controls the upgrade and cancellation flows. Tier checks are enforced by `UserCoreService.getTier()` inside each gated feature — not centrally.

Expected placement: Settings screen Account section, and paywall moments within gated features.

---

## Tiers

|Feature|Free|Pro ($3.99/mo)|Premium ($7.99/mo)|
|---|---|---|---|
|Active commitments|Up to 7|Unlimited|Unlimited|
|Storage|Local only|Firebase sync|Firebase sync|
|Devices|1|2|Unlimited|
|Daily encouragement|✓|✓|✓|
|Micro-insights|3/week|Unlimited|Unlimited|
|Weekly review|—|✓|✓|
|Manual deep report|—|✓|✓|
|Warning notifications|✓|✓|✓|
|Home screen widget|✓|✓|✓|
|Share progress|✓|✓|✓|

The `UserCapabilityService` gates (consistency + rate limit) apply equally to all tiers. The only tier difference for commitment creation is the portfolio ceiling.

---

## Upgrade Flow

1. User hits a paywall moment — commitment ceiling, micro-insight quota, or weekly report gate
2. Upgrade screen shown via RevenueCat
3. Payment confirmed → `subscriptionTier` updated in `UserCoreProfile`
4. If upgrading from free → `MigrationService.migrateToFirestore()` triggered automatically

**Best paywall moment:** free user hits micro-insight quota and taps "Understand this pattern" — highest intent, most motivated to upgrade.

---

## Downgrade / Cancellation Flow

1. User cancels via Settings → Account
2. `MigrationService.migrateToDrift()` runs — all data migrated to local Drift
3. All Firebase user data deleted — nothing remains in cloud
4. `storageBackend` switched to local
5. User retains full history locally and drops to free tier limits

Migration must complete before switching storage backend — never leave user in inconsistent state.

---

## Phase 1 Behavior

All users treated as free tier before RevenueCat is integrated. Tier gates enforced in code from day one — no feature needs reworking in Phase 3, only the payment flow is added.

---

## Dependencies

- `UserCoreService.getTier()` — read by any gated feature
- `MigrationService` — triggered on upgrade and cancellation
- RevenueCat — Phase 3 only