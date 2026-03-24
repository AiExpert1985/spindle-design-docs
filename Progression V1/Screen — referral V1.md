
**Created**: 17-Mar-2026 
**Modified**: - 
**Feature**: Progression 
**Phase**: 3

**Availability:** Pro / Premium · Level 2 (Journeyman) and above only.

**Standalone:** yes — entirely additive, no changes to other screens.

**Access:** Progression Screen share button only.

**Purpose:** lets users share their achievement as a visual card. The referral link embedded in the card gives the installer a first-month discount. The reward activates only after the referred user earns their first cup — preventing fake account gaming.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  SHARE YOUR ACHIEVEMENT              │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  Spindle                        │    │
│  │                                 │    │
│  │  Artisan                        │    │
│  │  32 pts  ·  🥉×8 🥈×5 🥇×3 💎×1│    │
│  │                                 │    │
│  │  "Consistent craft,             │    │
│  │   recognizable work."           │    │
│  │                                 │    │
│  │  weave your life, one thread    │    │
│  │  at a time. spindle.app         │    │
│  └─────────────────────────────────┘    │
│                                         │
│  [ Share ]                              │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  YOUR REFERRALS                         │
│  3 people downloaded via your link      │
│  2 earned their first cup               │
│                                         │
│  Anyone who downloads via your link     │
│  gets 30% off their first month.        │
│                                         │
└─────────────────────────────────────────┘
```

---

## The Achievement Card

A rendered widget captured as an image using `RepaintBoundary`. Contains:

- App name and tagline
- User's current level name and one-line flavor text
- Cup breakdown (counts per type)
- Total points
- Referral link embedded as a URL (not a QR code — simpler, works on all platforms)

The card is generated fresh each time the screen opens — always reflects current level and points.

---

## Share Behavior

Tap `[ Share ]` → native share sheet invoked with the card image and referral URL as text. User sees WhatsApp, Telegram, and any other installed app. No backend call needed for sharing itself — the referral link contains the user's referral code.

---

## Referral Mechanics

- Each Pro/Premium user has a permanent referral code stored on their `UserProfile` — generated once at account creation
- The referral link format: `spindle.app/join?ref={code}`
- When someone installs via the link, they are offered 30% off their first month at the paywall
- The discount is applied via RevenueCat promotional offers — no custom backend needed
- The referral is considered **activated** only after the referred user earns their first cup (one full week of good performance at ≥ 60%)
- Activated referral count is stored on the user's `UserProfile` — incremented via a Firebase Cloud Function when the referred user's first cup is written

---

## Referral Rewards (future consideration — not Phase 3)

The current system gives the _receiver_ a discount. Giving the _sharer_ a reward (e.g. subscription credit) requires tracking activated referrals server-side and applying credits via RevenueCat. This is deferred — the infrastructure (referral code, activation tracking) is built in Phase 3 but the sharer reward is left for a later phase.

When the sharer reward is added, no UI change is needed here — just a new row in the referral summary showing credit earned.

---

## Data Sources

|Data|Source|
|---|---|
|Current level and points|`ProgressionService.getProgressionSummary()`|
|Referral code|`UserProfile.referralCode`|
|Activated referral count|`UserProfile.activatedReferralCount`|
|Referral unlock status|`ProgressionService.isReferralUnlocked()`|

---

## Rules

- Screen is unreachable for free users and users below level 2 — `ProgressionService.isReferralUnlocked()` is checked before navigation, not just inside this screen
- The achievement card always reflects the user's current state — never cached
- No backend call is made from this screen — sharing is local, referral tracking happens server-side via Cloud Functions when the referred user acts