
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: User
**Phase**: 3
**Standalone:** yes — tier checks happen inside each gated feature, not in a central place.

**Purpose:** defines what each tier includes and controls access to gated features. 

---

## Tiers

| Feature | Free | Pro ($3.99/mo) | Premium ($7.99/mo) |
|---|---|---|---|
| Active commitments | Up to 7 | Unlimited | Unlimited |
| Storage | Local only | Firebase sync | Firebase sync |
| Devices | 1 | 2 | Unlimited |
| Daily encouragement | ✓ | ✓ | ✓ |
| Micro-insights | 3/week | Unlimited | Unlimited |
| Weekly review | — | ✓ | ✓ |
| Manual report | — | ✓ | ✓ |
| Notification actions | ✓ | ✓ | ✓ |
| Home screen widget | ✓ | ✓ | ✓ |
| Share progress | ✓ | ✓ | ✓ |

The unlock engine (consistency gate + rate limit) applies to all tiers equally. The only tier difference is the commitment ceiling.

---

## Upgrade Flow

1. User hits a paywall moment (commitment ceiling, micro-insight quota, weekly report gate)
2. Upgrade screen shown via RevenueCat
3. Payment confirmed → `subscriptionTier` updated in UserProfile
4. If upgrading from free → migration to Firestore triggered automatically

**Best paywall moment:** free user hits micro-insight quota and taps "Understand this pattern" — highest intent.

---

## Downgrade / Cancellation Flow

1. User cancels via Settings → Subscription
2. All Firestore data migrated to local Drift
3. All Firebase user data deleted
4. `storageBackend` switched to local in UserProfile
5. User retains full history locally, drops to free tier limits

Migration must complete successfully before switching storage backend — never leave user in inconsistent state.

---

## Phase 1 Behavior

Before RevenueCat is integrated, all users are treated as free tier. Tier gates are enforced in code from day one so no feature needs to be reworked in Phase 3 — only the payment flow is added.

---

## Dependencies

- `UserProfile.subscriptionTier` — read by any gated feature to check access
- RevenueCat — Phase 3 only
- Migration service — triggered on upgrade and cancellation
