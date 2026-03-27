**File Name**: service_temporal_helper **Feature**: TemporalHelper **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** answers semantic questions about time using the current user's preferences, and publishes domain-level time boundary events that any feature can subscribe to. Owns all time boundary detection in the app — no other feature detects day or week boundaries independently.

---

## Why TemporalHelper Is a Service, Not a Utility

If TemporalHelper were a static utility, every feature that needs time boundary detection would have to call `UserCoreService.getTemporalPreferences()` itself before calling it. That scatters a common dependency across every time-sensitive feature, and means any change to the preferences structure touches every caller.

As a service, TemporalHelper owns the UserCore dependency in one place. Callers pass a timestamp and get an answer — no preferences handling, no UserCore calls in their code. The service caches preferences internally and refreshes via `UserCoreService.watchProfile()` when preferences change.

---

## Why TemporalHelper Publishes Boundary Events

Multiple features need to react when a day or week ends — Cups, Rewards, Garment, AI Insights, and others. The naive approach is for each feature to subscribe to Heartbeat and call `isWeekBoundary()` independently. This scatters boundary detection across the system, duplicates TickGuard logic, and forces every time-sensitive feature to depend on Heartbeat directly.

TemporalHelper already owns boundary detection. Publishing the events here means one detection, one TickGuard check, and clean decoupling — features subscribe to a semantic signal (`WeekEndedEvent`) rather than a raw mechanism (Heartbeat tick). This also removes the coupling that previously forced CommitmentIdentityService to publish week events despite them being a time concern, not a commitment concern.

---

## Why TemporalHelper Is Separate From Heartbeat

Heartbeat's only job is to fire at intervals and emit a raw timestamp. It has no knowledge of what that timestamp means for any particular user.

TemporalHelper is where that meaning lives — and that meaning is user-specific. Week start, rest days, and day boundaries vary by region and preference. A user in Iraq has a Friday–Saturday weekend. A user in Europe has a Saturday–Sunday weekend. Heartbeat cannot know these things without depending on user preferences, which would give it domain knowledge it must not have.

---

## Position in the Stack

TemporalHelper sits above UserCore and below all domain features. It depends on `UserCoreService` and `Heartbeat`. Features above it subscribe to its boundary events or call its query functions downward.

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

Published via TickGuard to prevent double-firing when multiple ticks arrive near a boundary.

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

  if (isDayBoundary(ts) &&
      _guard.shouldRun('day_boundary', currentDayStart(ts))) {
    _guard.markRan('day_boundary', currentDayStart(ts));
    _publish(DayEndedEvent(dayEnd: ts));
    _publish(DayStartedEvent(dayStart: currentDayStart(ts)));
  }

  if (isWeekBoundary(ts) &&
      _guard.shouldRun('week_boundary', currentWeekStart(ts))) {
    _guard.markRan('week_boundary', currentWeekStart(ts));
    _publish(WeekEndedEvent(weekStart: previousWeekStart(ts)));
    _publish(WeekStartedEvent(weekStart: currentWeekStart(ts)));
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

## What TemporalHelper Does Not Do

- Does not know anything about commitments, instances, or any domain concept
- Does not make decisions — only answers questions about time and fires boundary signals

---

## Dependencies

- `Heartbeat` — subscribes to `longIntervalTick`
- `UserCoreService` — reads and watches temporal preferences
- `TickGuard` — prevents double-firing of boundary events

---

## Rules

- Every service that needs time-based condition checks calls `TemporalHelperService` — never raw timestamp arithmetic
- Every feature that needs to react to day or week boundaries subscribes to `TemporalHelperService` events — never detects boundaries independently
- No domain knowledge — never imports any feature model or event above it in the stack
- Preferences are cached — never read from `UserCoreService` on every function call