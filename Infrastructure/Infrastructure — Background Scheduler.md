**Created**: 15-Mar-2026 **Modified**: 26-Mar-2026 **Phase**: 1 **Feature**: Infrastructure

**Purpose:** a clock with two interval tiers. Publishes `LongIntervalTickEvent` and `ShortIntervalTickEvent` carrying only a raw timestamp. Knows nothing about what the timestamp means, who subscribes, or what any subscriber will do. It fires and steps back.

---

## Why Scheduler and TemporalHelper Are Separate

The scheduler's job is purely mechanical — fire at intervals, publish a timestamp. Interpreting what that timestamp means (is it midnight? is it the week boundary? is it waking hours?) is a separate concern that depends on the user's region and personal preferences. A user in Iraq has a different week start than a user in Europe. Midnight means the same clock time everywhere but a different cultural boundary depending on configuration.

Merging interpretation into the scheduler would require it to know about user preferences — making it feature-aware and breaking the infrastructure rule. Keeping them separate means the scheduler has exactly one responsibility: producing ticks. `TemporalHelper` has exactly one responsibility: giving ticks cultural meaning. See `infrastructure_temporal_helper`.

---

## Design

```
BackgroundScheduler  →  TickService  →  EventBus  →  subscribers
```

`BackgroundScheduler` is the platform adapter — it receives WorkManager callbacks and calls `TickService.onFire(timestamp)`. `TickService` creates the event and publishes it on the `EventBus`. Subscribers never know about WorkManager or any platform detail.

---

## TickService Abstraction

```dart
abstract class TickService {
  Stream<LongIntervalTickEvent> get longIntervalTick;
  Stream<ShortIntervalTickEvent> get shortIntervalTick;
  void onFire(DateTime timestamp);
  DateTime? getLastTickTimestamp();
}
```

Both events carry only a timestamp:

```dart
class LongIntervalTickEvent extends AppEvent {
  final DateTime timestamp;
}

class ShortIntervalTickEvent extends AppEvent {
  final DateTime timestamp;
}
```

### `getLastTickTimestamp() → DateTime?`

Returns the timestamp of the last tick fired, or null on first ever launch. Persisted to local storage on every tick before publishing. Used by any feature on app open to detect missed boundaries while the app was closed — the feature reads this value and processes any missed intervals through its normal tick handler using TickGuard for idempotency.

---

## TickGuard

Shared idempotency utility for all tick subscribers. Prevents double-execution when the OS fires two ticks in the same period or when catch-up ticks replay missed intervals.

```dart
class TickGuard {
  bool shouldRun(String actionKey, dynamic period);
  void markRan(String actionKey, dynamic period);
}
```

`period` is a `Date` for daily actions, week-start `DateTime` for weekly, truncated `DateTime` for within-day actions. State held in memory — resets on app restart. Safe because all tick operations are idempotent at the data layer.

**Every tick subscriber must use TickGuard. Never implement custom idempotency.**

---

## Phase 1 Implementation

One WorkManager periodic task, ~15 minutes. Both streams receive the same tick.

```dart
class DefaultTickService implements TickService {
  final _controller = StreamController<TickEvent>.broadcast();
  DateTime? _lastTimestamp;

  @override
  Stream<LongIntervalTickEvent> get longIntervalTick =>
      _controller.stream.map((e) => LongIntervalTickEvent(e.timestamp));

  @override
  Stream<ShortIntervalTickEvent> get shortIntervalTick =>
      _controller.stream.map((e) => ShortIntervalTickEvent(e.timestamp));

  @override
  DateTime? getLastTickTimestamp() => _lastTimestamp;

  @override
  void onFire(DateTime now) {
    _lastTimestamp = now;
    _persistLastTimestamp(now);   // local storage write
    _controller.add(TickEvent(now));
  }
}
```

`shortIntervalTick` subscribers receive 15-minute ticks in Phase 1. This is acceptable — window warnings firing up to 15 minutes late is tolerable for MVP. Phase 2 implementation separates them.

---

## Phase 2 Implementation

Two separate WorkManager tasks when short-interval accuracy matters.

```dart
class DualTickService implements TickService {
  final _long  = StreamController<LongIntervalTickEvent>.broadcast();
  final _short = StreamController<ShortIntervalTickEvent>.broadcast();
  DateTime? _lastTimestamp;

  @override
  Stream<LongIntervalTickEvent>  get longIntervalTick  => _long.stream;
  @override
  Stream<ShortIntervalTickEvent> get shortIntervalTick => _short.stream;
  @override
  DateTime? getLastTickTimestamp() => _lastTimestamp;

  void onLongFire(DateTime now)  {
    _lastTimestamp = now;
    _persistLastTimestamp(now);
    _long.add(LongIntervalTickEvent(now));
  }
  void onShortFire(DateTime now) => _short.add(ShortIntervalTickEvent(now));
}
```

Swapping from `DefaultTickService` to `DualTickService` requires changing one dependency injection binding. Every subscriber is completely unaffected.

---

## App Open Catch-Up

On app open, any feature that needs to process missed boundaries calls `getLastTickTimestamp()` and compares against the current time. For each missed interval it calls its own tick handler with the historical timestamp. TickGuard prevents double-work — all catch-up processing uses the same idempotency path as live ticks.

Short interval catch-up is not replayed — only long interval matters. Missed short-interval checks are harmless to skip.

---

## Platform Accuracy

WorkManager does not guarantee exact intervals. Long interval: expect 10–20 minutes in practice. Short interval: expect 1–3 minutes. This is acceptable — `TemporalHelper` conditions are checked on each tick arrival regardless of exact timing.

---

## Rules

- `TickService` is the abstraction — never reference `DefaultTickService` or `DualTickService` outside dependency injection
- Tick events carry timestamp only — no pre-computed booleans, no feature knowledge
- Subscribers use `TemporalHelper` for all time-based condition checks — never raw timestamp comparisons
- Subscribers use `TickGuard` for idempotency — never custom implementations
- The scheduler never calls any feature service — it only fires ticks
- `getLastTickTimestamp()` returns null on first launch — callers must handle null
- Adding a new time-based behavior requires zero changes to the scheduler