**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Settings **Phase**: 1 (temporal) · Phase 2 (notifications)

**Purpose:** reads and writes user preferences. Thin layer over UserRepository — no complex logic. Pure Dart.

---

## Temporal Functions (Phase 1)

### `getTemporalPreferences()`

Returns current temporal settings: `weekStartDay`, `restDays`, `dayBoundaryHour`, `wakingHoursStart`, `wakingHoursEnd`. Called by `TemporalHelper` to read user preferences.

### `updateWeekStartDay(day)`

Updates `weekStartDay` in UserProfile. Takes effect immediately via `TemporalHelper`.

### `updateRestDays(days)`

Updates `restDays` list in UserProfile.

### `updateDayBoundaryHour(hour)`

Updates `dayBoundaryHour` in UserProfile.

### `updateWakingHours(start, end)`

Updates `wakingHoursStart` and `wakingHoursEnd` in UserProfile.

---

## Notification Functions (Phase 2)

### `getPreferences()`

Returns full UserProfile preferences. Called when settings screen opens.

### `updateCelebrationSettings(enabled, time, threshold)`

Updates day celebration settings in UserProfile.

### `updateWeeklyReportSettings(enabled, time)`

Updates weekly report settings in UserProfile.

### `updateWarningNotifications(enabled)`

Toggles warning notifications globally.

---

## Rules

- Every function reads or writes UserProfile through `UserRepository` only
- No function contains business logic — validation lives in the UI layer
- Temporal preference changes do not require rescheduling anything — `TemporalHelper` reads live from UserProfile
- Storage backend changes belong to `MigrationService` — not here
- `TemporalHelper` calls `getTemporalPreferences()` — `SettingsService` is the only path to temporal preferences