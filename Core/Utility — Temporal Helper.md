**File Name**: temporalhelper **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026 **Phase**: 1 **Feature**: Core

**Purpose:** answers semantic questions about time using the user's cultural and personal preferences. Any service that needs to check whether a timestamp represents a meaningful boundary — day start, week start, rest day, waking hours — calls this helper. The scheduler fires ticks with raw timestamps. This helper gives those timestamps meaning.

---

## Why This Is Separate From the Background Scheduler

The scheduler's only job is to fire at intervals and publish a raw timestamp. It has no knowledge of what that timestamp means for any particular user.

`TemporalHelper` is where that meaning lives — and that meaning is culturally relative. Week start, rest days, and day boundaries all vary by region and personal preference. The scheduler cannot know these things without depending on user preferences, which would make it feature-aware and break the infrastructure rule.

Keeping them separate gives each component exactly one responsibility: the scheduler produces ticks, `TemporalHelper` interprets them. Any subscriber that needs to act on a time boundary calls `TemporalHelper` with the tick's timestamp — the scheduler is never involved in that decision.

---

## Interface

```dart
abstract class TemporalHelper {

  // Is this timestamp at or within the user's configured day boundary?
  // Day boundary = the hour at which the "day" resets (default midnight)
  bool isDayBoundary(DateTime timestamp);

  // Is this timestamp at or within the user's configured week boundary?
  // Week boundary = first day boundary after the last day of the user's week
  bool isWeekBoundary(DateTime timestamp);

  // Is this a rest day for this user?
  // Based on user's configured restDays list
  bool isRestDay(DateTime timestamp);

  // Is this timestamp within waking hours for this user?
  // Based on user's configured wakingHoursStart and wakingHoursEnd
  bool isWakingHours(DateTime timestamp);

  // Returns the start of the current day for this user
  // Uses dayBoundaryHour from preferences
  DateTime currentDayStart(DateTime timestamp);

  // Returns the start of the current week for this user
  // Uses weekStartDay from preferences
  DateTime currentWeekStart(DateTime timestamp);

  // Returns the start of the previous week for this user
  DateTime previousWeekStart(DateTime timestamp);

  // Is this the first tick-firing after the week boundary?
  // Used by subscribers that need to act once per week
  bool isFirstTickOfWeek(DateTime timestamp);

  // Is this the first tick-firing after the day boundary?
  // Used by subscribers that need to act once per day
  bool isFirstTickOfDay(DateTime timestamp);
}
```

---

## Implementation

`DefaultTemporalHelper` reads preferences from `UserCoreService.getTemporalPreferences()` on each call. Preferences are cached in memory and refreshed when `UserCoreProfile` changes — no repeated database reads on every tick.

---

## User Preferences Used

All read from `UserCoreProfile` via `UserCoreService`:

|Preference|Default|Notes|
|---|---|---|
|`weekStartDay`|1 (Monday)|0=Sunday, 1=Monday, 5=Friday, 6=Saturday|
|`restDays`|[6, 7] (Sat, Sun)|List of integers. Middle East default: [5, 6] (Fri, Sat)|
|`dayBoundaryHour`|0 (midnight)|Hour at which the day resets. 0–23.|
|`wakingHoursStart`|7|Hour when waking hours begin|
|`wakingHoursEnd`|22|Hour when waking hours end|

---

## Usage Pattern

```dart
// CommitmentIdentityService — correct usage
eventBus.on<LongIntervalTickEvent>().listen((tick) {
  if (_temporal.isDayBoundary(tick.timestamp) &&
      _guard.shouldRun('day_boundary', _temporal.currentDayStart(tick.timestamp))) {
    _guard.markRan('day_boundary', _temporal.currentDayStart(tick.timestamp));
    generateUpcomingInstances();
  }

  if (_temporal.isWeekBoundary(tick.timestamp) &&
      _guard.shouldRun('week_boundary', _temporal.currentWeekStart(tick.timestamp))) {
    _guard.markRan('week_boundary', _temporal.currentWeekStart(tick.timestamp));
    eventBus.publish(WeekEndedEvent(_temporal.previousWeekStart(tick.timestamp)));
    eventBus.publish(WeekStartedEvent(_temporal.currentWeekStart(tick.timestamp)));
  }

  if (_temporal.isWakingHours(tick.timestamp)) {
    evaluateWindowWarnings(tick.timestamp);
  }
});
```

---

## What TemporalHelper Does NOT Do

- Does not know anything about commitments, windows, or business logic
- Does not publish events
- Does not subscribe to anything
- Does not make decisions — only answers questions about time
- Does not cache results across calls — each call reads current preferences

---

## Rules

- Every service that needs time-based condition checks uses `TemporalHelper` — never hardcoded time comparisons
- `TemporalHelper` is injected like any other infrastructure dependency
- Preferences are read via `UserCoreService` — `TemporalHelper` never reads `UserCoreRepository` directly
- Default values must be sensible for the majority of users — Monday week start, midnight day boundary, Saturday + Sunday rest days
- Services never store `TemporalHelper` results — always call fresh on each tick

---

## Dependencies

- `UserCoreService` — reads temporal preferences from `UserCoreProfile`