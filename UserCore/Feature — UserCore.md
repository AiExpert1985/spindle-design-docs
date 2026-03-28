**File Name**: feature_usercore **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

UserCore is the single source of truth for the user's identity, subscription tier, and all low-level preferences that the rest of the app reads to function.

---

## Why It Exists

Every feature above it needs something from the user's profile — tier to gate behavior, temporal preferences to interpret time, notification toggles, activity window defaults. Without a dedicated low-level feature for this data, every feature that needs it would depend on wherever it happened to live, creating either an impossible dependency chain or forced cycles. UserCore solves this by sitting as low as possible — just above Infrastructure — so any feature above it can read freely without creating an upward dependency.

---

## Position in the System

Sits just above Infrastructure. Depends only on Infrastructure services. Everything above it may read from it. It publishes no events — profile changes propagate via a live stream that any feature can watch.

Read-only from the outside. The only writer of `UserCoreProfile` is the feature at the top of the chain responsible for user-facing settings — it writes directly through the repository. `UserCoreService` never writes after initial creation.

---

## What It Owns

**Model:** `UserCoreProfile` — identity, subscription tier, storage backend, temporal preferences, activity window defaults, notification preferences.

**Service:** `UserCoreService` — read-only access and first-launch profile creation.

**Repository:** `UserCoreRepository` — storage interface for `UserCoreProfile`. Drift in Phase 1, Firestore in Phase 3.

UserCore is the single source of truth for all data in `UserCoreProfile`. No other feature stores or duplicates these values.

---

## How It Works

On first launch, `UserCoreService` finds no profile in storage and creates one with defaults derived from the user's selected region. After that, the profile is only changed through the settings feature. Every other feature reads the profile through `UserCoreService` — either a one-time read or a live stream for features that cache preferences and need to invalidate when settings change.

Temporal preferences are the most-read data in the profile. The rule is strict: no feature reads them directly from `UserCoreService`. The feature responsible for time interpretation owns that dependency and is the only caller — all others go through it.

**Key design decisions**

- Read-only service, write-only from the top — the split enforces that reading preferences never requires knowing anything about commitments, performance, or behavior. That knowledge lives higher.
- No events published — `watchProfile()` stream is sufficient. Profile changes are infrequent and always initiated by the user, not by system logic.
- Full replace on every write — no partial field updates. Simpler to reason about, no partial-state bugs.

---

## Who Uses It

Any feature that needs to gate behavior on subscription tier calls `getTier()`. Any feature that needs activity window defaults when creating a commitment calls `getActivityWindowDefaults()`. The time interpretation feature watches the profile stream to invalidate its cached preferences when the user changes their region or time settings. No other feature reads temporal preferences directly.

---

## Rules

- No feature reads temporal preferences from `UserCoreService` directly — those go through the time interpretation feature above it
- `storageBackend` is written only by `MigrationService` — no other service touches this field
- One profile per user — created once on first launch, never recreated
- Defaults are applied once at creation — never re-applied after the user changes a setting

---

## Later Improvements

No planned extensions. UserCore is intentionally minimal — new user-level state that requires behavioral logic goes into `UserSettingsProfile`, not here.