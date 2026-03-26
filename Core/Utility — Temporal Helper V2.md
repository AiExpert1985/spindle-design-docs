**File Name**: temporal_helper **Layer**: Domain **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** a static utility class that answers semantic questions about time using the user's preferences. Any service that needs to check whether a timestamp represents a meaningful boundary — end of day, end of week, rest day, waking hours — calls this class directly. It takes a timestamp and a preferences object, and returns an answer. It knows nothing about features, events, or ticks.

---

## Why TemporalHelper Is Separate From Heartbeat

Heartbeat's only job is to fire at intervals and emit a raw timestamp. It has no knowledge of what that timestamp means for any particular user.

TemporalHelper is where that meaning lives — and that meaning is user-specific. Week start, rest days, and day boundaries all vary by region and personal preference. A user in Iraq has a Friday–Saturday weekend. A user in Europe has a Saturday–Sunday weekend. Heartbeat cannot know these things without depending on user preferences, which would give it domain knowledge it must not have.

Keeping them separate gives each exactly one responsibility: Heartbeat produces timestamps, TemporalHelper interprets them. A service that receives a tick passes the timestamp to TemporalHelper along with the user's preferences — Heartbeat is never involved in that interpretation.

---

## Design

TemporalHelper is a plain Dart class with static functions. It has no state, no storage, no events, and no lifecycle. It is not a feature and is not injected — any service imports and calls it directly.

It lives in the Domain layer: pure Dart, no dependencies, available to everything above it.

```dart
class TemporalHelper {
  static bool isDayBoundary(DateTime timestamp, UserTemporalPrefs prefs) { ... }
  static bool isWeekBoundary(DateTime timestamp, UserTemporalPrefs prefs) { ... }
  static bool isRestDay(DateTime timestamp, UserTemporalPrefs prefs) { ... }
  static bool isWakingHours(DateTime timestamp, UserTemporalPrefs prefs) { ... }
  static DateTime currentDayStart(DateTime timestamp, UserTemporalPrefs prefs) { ... }
  static DateTime currentWeekStart(DateTime timestamp, UserTemporalPrefs prefs) { ... }
  static DateTime previousWeekStart(DateTime timestamp, UserTemporalPrefs prefs) { ... }
}
```

---

## Functions

`isDayBoundary(timestamp, prefs) → bool` Returns true if the timestamp falls at or within the user's configured day boundary — the hour at which the day resets. Default: midnight.

`isWeekBoundary(timestamp, prefs) → bool` Returns true if the timestamp falls at or within the first day boundary after the last day of the user's configured week.

`isRestDay(timestamp, prefs) → bool` Returns true if the timestamp falls on a day the user has configured as a rest day.

`isWakingHours(timestamp, prefs) → bool` Returns true if the timestamp falls within the user's configured waking hours window.

`currentDayStart(timestamp, prefs) → DateTime` Returns the start of the current day for this user, based on their configured day boundary hour.

`currentWeekStart(timestamp, prefs) → DateTime` Returns the start of the current week for this user, based on their configured week start day.

`previousWeekStart(timestamp, prefs) → DateTime` Returns the start of the previous week for this user.

---

## User Preferences Used

All passed in via `UserTemporalPrefs` — a plain value object the caller assembles from user settings.

|Preference|Default|Notes|
|---|---|---|
|`weekStartDay`|1 (Monday)|0=Sunday, 1=Monday, 5=Friday, 6=Saturday|
|`restDays`|[6, 7] (Sat, Sun)|Middle East default: [5, 6] (Fri, Sat)|
|`dayBoundaryHour`|0 (midnight)|Hour at which the day resets. 0–23.|
|`wakingHoursStart`|7|Hour when waking hours begin|
|`wakingHoursEnd`|22|Hour when waking hours end|

---

## Usage Pattern

```dart
// A service subscribing to Heartbeat ticks
_heartbeat.longIntervalTick.listen((tick) {
  final prefs = _userSettingsService.getTemporalPrefs();

  if (TemporalHelper.isDayBoundary(tick.timestamp, prefs) &&
      _guard.shouldRun('day_boundary', TemporalHelper.currentDayStart(tick.timestamp, prefs))) {
    _guard.markRan('day_boundary', TemporalHelper.currentDayStart(tick.timestamp, prefs));
    _onDayBoundary();
  }

  if (TemporalHelper.isWeekBoundary(tick.timestamp, prefs) &&
      _guard.shouldRun('week_boundary', TemporalHelper.currentWeekStart(tick.timestamp, prefs))) {
    _guard.markRan('week_boundary', TemporalHelper.currentWeekStart(tick.timestamp, prefs));
    _onWeekBoundary(TemporalHelper.previousWeekStart(tick.timestamp, prefs));
  }
});
```

---

## What TemporalHelper Does Not Do

- Does not know anything about commitments, windows, or any domain concept
- Does not publish or subscribe to events
- Does not make decisions — only answers questions about time
- Does not read preferences itself — the caller passes them in

---

## Rules

- Every service that needs time-based condition checks uses `TemporalHelper` — never raw timestamp arithmetic
- The caller is responsible for reading preferences and passing them in — `TemporalHelper` never reads from any service or repository
- Default preference values must be sensible for the majority of users — Monday week start, midnight day boundary, Saturday + Sunday rest days