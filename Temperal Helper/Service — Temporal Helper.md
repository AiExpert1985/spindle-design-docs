**File Name**: service_temporal_helper **Feature**: TemporalHelper **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** answers semantic questions about time using the current user's preferences, and publishes boundary events when a day or week ends or begins. Preferences are owned by `UserCoreService` and cached internally — callers pass a timestamp and get an answer, no preferences handling required.

---

## Events Published

```
DayStartedEvent
  dayStart: DateTime      // start of the new day per user's dayBoundaryHour

DayEndedEvent
  dayEnd: DateTime        // end of the day that just closed

WeekEndedEvent
  weekStart: DateTime     // start of the week that just ended

WeekStartedEvent
  weekStart: DateTime     // start of the new week
```

Published when a boundary is detected on a long-interval tick. Deduplication is handled internally — `_lastPublishedDayStart` and `_lastPublishedWeekStart` are stored in memory and compared on each tick to prevent double-firing within a session.

---

## Query Interface

```dart
abstract class TemporalHelperService {
  // Query functions — callable by any feature
  bool isDayBoundary(DateTime timestamp);
  bool isWeekBoundary(DateTime timestamp);
  bool isRestDay(DateTime timestamp);
  bool isWakingHours(DateTime timestamp);
  DateTime currentDayStart(DateTime timestamp);
  DateTime currentWeekStart(DateTime timestamp);
  DateTime previousWeekStart(DateTime timestamp);

  // Boundary event streams — subscribable by any feature
  Stream<DayStartedEvent> get onDayStarted;
  Stream<DayEndedEvent> get onDayEnded;
  Stream<WeekEndedEvent> get onWeekEnded;
  Stream<WeekStartedEvent> get onWeekStarted;
}
```

---

## Query Functions

`isDayBoundary(timestamp) → bool` Returns true if the timestamp falls at or within the user's configured day boundary — the hour at which the day resets.

`isWeekBoundary(timestamp) → bool` Returns true if the timestamp falls at or within the first day boundary after the last day of the user's configured week.

`isRestDay(timestamp) → bool` Returns true if the timestamp falls on a day the user has configured as a rest day.

`isWakingHours(timestamp) → bool` Returns true if the timestamp falls within the user's configured waking hours window.

`currentDayStart(timestamp) → DateTime` Returns the start of the current day for this user, based on their configured day boundary hour.

`currentWeekStart(timestamp) → DateTime` Returns the start of the current week for this user, based on their configured week start day.

`previousWeekStart(timestamp) → DateTime` Returns the start of the previous week for this user.

---

## Heartbeat Subscription

```dart
_heartbeat.longIntervalTick.listen((tick) {
  final ts = tick.timestamp;
  final dayStart = currentDayStart(ts);
  final weekStart = currentWeekStart(ts);

  if (isDayBoundary(ts) && dayStart != _lastPublishedDayStart) {
    _lastPublishedDayStart = dayStart;
    _publish(DayEndedEvent(dayEnd: ts));
    _publish(DayStartedEvent(dayStart: dayStart));
  }

  if (isWeekBoundary(ts) && weekStart != _lastPublishedWeekStart) {
    _lastPublishedWeekStart = weekStart;
    _publish(WeekEndedEvent(weekStart: previousWeekStart(ts)));
    _publish(WeekStartedEvent(weekStart: weekStart));
  }
});
```

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

## Dependencies

- `Heartbeat` — subscribes to `longIntervalTick`
- `UserCoreService` — reads and watches temporal preferences

---

## Rules

- No domain knowledge — never imports any feature model or event above it in the stack
- Preferences are cached — never read from `UserCoreService` on every function call