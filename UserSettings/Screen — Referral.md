**File Name**: screen_referral **Feature**: UserSettings **Phase**: 3 **Created**: 17-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** lets users share their achievement as a visual card. The referral link embedded in the card gives the installer a first-month discount. The reward activates only after the referred user earns their first cup — preventing fake account gaming.

Accessed from the Progression screen share button only. Not directly navigable. Available to Pro / Premium users at level 2 (Journeyman) and above.

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

## Achievement Card

A rendered Flutter widget captured as an image using `RepaintBoundary`. Contains: app name and tagline, user's current level name and one-line flavor text, cup breakdown (counts per type), total points, and referral link as a URL.

Generated fresh each time the screen opens — always reflects current level and points. Never cached.

---

## Share Behavior

Tap `[ Share ]` → native Android share sheet with the card image and referral URL. No backend call needed — the referral code is embedded in the link. The user shares via WhatsApp, Telegram, or any installed app.

Referral link format: `spindle.app/join?ref={code}`

---

## Referral Mechanics

- Each Pro / Premium user has a permanent referral code on their `UserSettingsProfile` — generated once at account creation
- When someone installs via the link, they receive 30% off their first month at the paywall — applied via RevenueCat promotional offers, no custom backend needed
- A referral is **activated** only after the referred user earns their first cup — one full week of performance at ≥ 60%
- `activatedReferralCount` stored on `UserSettingsProfile` — incremented via Firebase Cloud Function when the referred user's first cup is written, never by the app directly

---

## Rules

- Screen is unreachable for free users and users below level 2 — gate checked before navigation in the Progression screen
- No backend call made from this screen — sharing is local, referral tracking happens server-side
- Card always reflects current state — never cached between opens

---

## Data Sources

|Data|Source|
|---|---|
|Current level and points|`ProgressionService.getProgressionSummary()`|
|Cup breakdown|`AchievementService.getCupHistory(from, to)`|
|Referral code|`UserSettingsProfile.referralCode` via `UserSettingsService`|
|Activated referral count|`UserSettingsProfile.activatedReferralCount` via `UserSettingsService`|
|Referral unlock status|`ProgressionService.isReferralUnlocked()`|

---

## Later Improvements

**Sharer reward.** Give the sharer subscription credit when a referred user activates. The infrastructure (referral code, activation tracking via Cloud Function) is already built in Phase 3. Adding the reward requires a new row in the referral summary and a RevenueCat credit call — no UI redesign needed.