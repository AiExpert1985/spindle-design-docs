**File Name**: service_temporal_helper **Feature**: TemporalHelper **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** answers semantic questions about time using the current user's preferences. Any service that needs to check whether a timestamp represents a meaningful boundary — end of day, end of week, rest day, waking hours — calls this service. Callers pass only a timestamp. TemporalHelper reads the user's preferences internally and returns the answer.

---

## Why TemporalHelper Is a Service, Not a Utility

If TemporalHelper were a static utility, every feature that needs time boundary detection would have to call `UserCoreService.getTemporalPreferences()` itself before calling it. That scatters a common dependency across every time-sensitive feature, and means any change to the preferences structure touches every caller.

As a service, TemporalHelper owns the UserCore dependency in one place. Callers pass a timestamp and get an answer — no preferences handling, no UserCore calls in their code. The service caches preferences internally and refreshes via `UserCoreService.watchProfile()` when preferences change.

---

## Why TemporalHelper Is Separate From Heartbeat

Heartbeat's only job is to fire at intervals and emit a raw timestamp. It has no knowledge of what that timestamp means for any particular user.

TemporalHelper is where that meaning lives — and that meaning is user-specific. Week start, rest days, and day boundaries vary by region and preference. A user in Iraq has a Friday–Saturday weekend. A user in Europe has a Saturday–Sunday weekend. Heartbeat cannot know these things without depending on user preferences, which would give it domain knowledge it must not have.

---

## Interface

```dart
abstract class TemporalHelperService {
  bool isDayBoundary(DateTime timestamp);
  bool isWeekBoundary(DateTime timestamp);
  bool isRestDay(DateTime timestamp);
  bool isWakingHours(DateTime timestamp);
  DateTime currentDayStart(DateTime timestamp);
  DateTime currentWeekStart(DateTime timestamp);
  DateTime previousWeekStart(DateTime timestamp);
}
```

---

## Functions

`isDayBoundary(timestamp) → bool` Returns true if the timestamp falls at or within the user's configured day boundary — the hour at which the day resets.

`isWeekBoundary(timestamp) → bool` Returns true if the timestamp falls at or within the first day boundary after the last day of the user's configured week.

`isRestDay(timestamp) → bool` Returns true if the timestamp falls on a day the user has configured as a rest day.

`isWakingHours(timestamp) → bool` Returns true if the timestamp falls within the user's configured waking hours window.

`currentDayStart(timestamp) → DateTime` Returns the start of the current day for this user, based on their configured day boundary hour.

`currentWeekStart(timestamp) → DateTime` Returns the start of the current week for this user, based on their configured week start day.

`previousWeekStart(timestamp) → DateTime` Returns the start of the previous week for this user.

---

## Preferences Used

All read internally from `UserCoreService.getTemporalPreferences()` and cached in memory.

|Preference|Default|Notes|
|---|---|---|
|`weekStartDay`|1 (Monday)|0=Sunday, 1=Monday, 5=Friday, 6=Saturday|
|`restDays`|[6, 7] (Sat, Sun)|Middle East default: [5, 6] (Fri, Sat)|
|`dayBoundaryHour`|0 (midnight)|Hour at which the day resets. 0–23.|
|`wakingHoursStart`|7|Hour when waking hours begin|
|`wakingHoursEnd`|22|Hour when waking hours end|

Preferences are cached on first call and refreshed when `UserCoreService.watchProfile()` emits a change. No repeated reads on every tick.

---

## Usage Pattern

```dart
// CommitmentIdentityService — subscribing to Heartbeat ticks
_heartbeat.longIntervalTick.listen((tick) {
  if (_temporal.isDayBoundary(tick.timestamp) &&
      _guard.shouldRun('day_boundary', _temporal.currentDayStart(tick.timestamp))) {
    _guard.markRan('day_boundary', _temporal.currentDayStart(tick.timestamp));
    _onDayBoundary();
  }

  if (_temporal.isWeekBoundary(tick.timestamp) &&
      _guard.shouldRun('week_boundary', _temporal.currentWeekStart(tick.timestamp))) {
    _guard.markRan('week_boundary', _temporal.currentWeekStart(tick.timestamp));
    _onWeekBoundary(_temporal.previousWeekStart(tick.timestamp));
  }
});
```

---

## What TemporalHelper Does Not Do

- Does not know anything about commitments, instances, or any domain concept
- Does not publish or subscribe to events
- Does not make decisions — only answers questions about time

---

## Dependencies

- `UserCoreService` — reads and watches temporal preferences

---

## Rules

- Every service that needs time-based condition checks calls `TemporalHelperService` — never raw timestamp arithmetic
- No domain knowledge — never imports any feature model or event above it in the stack
- Preferences are cached — never read from `UserCoreService` on every function call