**File Name**: service_user_core **Feature**: UserCore **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** read-only access to `UserCoreProfile` for any feature that needs it. Owns profile creation on first launch. Exposes no write functions — `UserSettingsService` is the only writer.

---

## Why UserCore Is a Separate Feature

Every feature in the app needs something from the user's profile — tier, temporal preferences, notification toggles, activity window defaults. If this data lived inside a higher-level feature, every feature that reads it would depend on that higher-level feature, creating an impossible chain or forcing cycles.

UserCore solves this by being the lowest domain feature: it depends on nothing above the base features, and everything above it can read from it freely. Writes flow from the top — `UserSettingsService` owns all preference updates and writes through `UserCoreRepository` directly. Reads flow from the bottom — any feature calls `UserCoreService` downward.

This split also enforces a clean rule: reading preferences never requires knowing anything about commitments, performance, or user behavior. That knowledge lives higher in the chain where it belongs.

---

## Functions

### `getProfile() → UserCoreProfile`

Returns the current profile. On first launch, when the repository returns null, creates the profile with defaults and persists it via `UserCoreRepository`. Never returns null to callers.

### `getTier() → SubscriptionTier`

Returns the current subscription tier. Called by any feature that gates behavior on tier.

### `getTemporalPreferences() → UserTemporalPrefs`

Returns `weekStartDay`, `restDays`, `dayBoundaryHour`, `wakingHoursStart`, `wakingHoursEnd` as a `UserTemporalPrefs` value object. Passed directly into `TemporalHelper` static functions by callers — `TemporalHelper` never calls this itself.

### `getActivityWindowDefaults(recurrence) → ActivityWindowDefaults`

Returns the default `startMinutes` and `durationMinutes` for a given recurrence type. Called when creating a new commitment to pre-fill the activity window.

### `watchProfile() → Stream<UserCoreProfile>`

Live stream of profile changes. Used by features that cache preferences and need to invalidate the cache when the user changes their settings.

---

## Dependencies

- `UserCoreRepository` — reads `UserCoreProfile`

---

## Rules

- No write functions — `UserSettingsService` writes via `UserCoreRepository` directly
- `storageBackend` changed only by `MigrationService` — `UserCoreService` never writes this field
- No events published — profile changes propagate via `watchProfile()` stream
- All temporal preference reads by any feature go through `TemporalHelperService` — no feature calls `getTemporalPreferences()` directly except `TemporalHelperService` itself