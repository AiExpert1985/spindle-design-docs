**File Name**: heartbeat **Feature**: Infrastructure **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the app's pulse. Fires periodic signals — a long-interval tick and a short-interval tick — carrying only a raw timestamp. Any feature that needs time-based behavior subscribes to the appropriate stream. Heartbeat knows nothing about what the timestamp means, who subscribes, or what any subscriber will do.

---

## Why Heartbeat and TemporalHelper Are Separate

Heartbeat's job is purely mechanical — fire at intervals, emit a timestamp. Interpreting what that timestamp means (is it midnight? is it the week boundary? is it within waking hours?) depends on the user's region and preferences. A user in Iraq has a different week start than a user in Europe.

Merging interpretation into Heartbeat would require it to know about user preferences — making it domain-aware and breaking the infrastructure rule. Keeping them separate means Heartbeat has exactly one responsibility: producing ticks. `TemporalHelper` has exactly one responsibility: giving ticks cultural meaning. See `infrastructure_temporal_helper`.

---

## Design

```
WorkManager  →  Heartbeat  →  TickService streams  →  subscribers
```

`Heartbeat` is the platform adapter — it receives WorkManager callbacks and calls `TickService.onFire(timestamp)`. `TickService` emits the timestamp on its streams. Subscribers never know about WorkManager or any platform detail.

---

## TickService Abstraction

```dart
abstract class TickService {
  Stream<LongIntervalTick> get longIntervalTick;
  Stream<ShortIntervalTick> get shortIntervalTick;
  void onFire(DateTime timestamp);
  DateTime? getLastTickTimestamp();
}
```

Both events carry only a timestamp:

```dart
class LongIntervalTick  { final DateTime timestamp; }
class ShortIntervalTick { final DateTime timestamp; }
```

### `getLastTickTimestamp() → DateTime?`

Returns the timestamp of the last tick fired, or null on first launch. Persisted to local storage on every tick before emitting. Used by any feature on app open to detect missed boundaries while the app was closed — the feature reads this value and processes any missed intervals through its normal tick handler using TickGuard for idempotency.

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
  final _controller = StreamController<DateTime>.broadcast();
  DateTime? _lastTimestamp;

  @override
  Stream<LongIntervalTick> get longIntervalTick =>
      _controller.stream.map((t) => LongIntervalTick(t));

  @override
  Stream<ShortIntervalTick> get shortIntervalTick =>
      _controller.stream.map((t) => ShortIntervalTick(t));

  @override
  DateTime? getLastTickTimestamp() => _lastTimestamp;

  @override
  void onFire(DateTime now) {
    _lastTimestamp = now;
    _persistLastTimestamp(now);
    _controller.add(now);
  }
}
```

`shortIntervalTick` subscribers receive 15-minute ticks in Phase 1. Window warnings firing up to 15 minutes late is acceptable for MVP. Phase 2 separates them.

---

## Phase 2 Implementation

Two separate WorkManager tasks when short-interval accuracy matters.

```dart
class DualTickService implements TickService {
  final _long  = StreamController<LongIntervalTick>.broadcast();
  final _short = StreamController<ShortIntervalTick>.broadcast();
  DateTime? _lastTimestamp;

  @override
  Stream<LongIntervalTick>  get longIntervalTick  => _long.stream;
  @override
  Stream<ShortIntervalTick> get shortIntervalTick => _short.stream;
  @override
  DateTime? getLastTickTimestamp() => _lastTimestamp;

  void onLongFire(DateTime now) {
    _lastTimestamp = now;
    _persistLastTimestamp(now);
    _long.add(LongIntervalTick(now));
  }
  void onShortFire(DateTime now) => _short.add(ShortIntervalTick(now));
}
```

Swapping from `DefaultTickService` to `DualTickService` requires changing one dependency injection binding. Every subscriber is unaffected.

---

## App Open Catch-Up

On app open, any feature that needs to process missed boundaries calls `getLastTickTimestamp()` and compares against the current time. For each missed interval it calls its own tick handler with the historical timestamp. TickGuard prevents double-work — catch-up uses the same idempotency path as live ticks.

Short interval catch-up is not replayed — only long interval matters. Missed short-interval checks are harmless to skip.

---

## Platform Accuracy

WorkManager does not guarantee exact intervals. Long interval: expect 10–20 minutes in practice. Short interval: expect 1–3 minutes. This is acceptable — `TemporalHelper` conditions are checked on each tick arrival regardless of exact timing.

---

## Rules

- `TickService` is the abstraction — never reference `DefaultTickService` or `DualTickService` outside dependency injection
- Tick events carry timestamp only — no pre-computed booleans, no domain knowledge
- Subscribers use `TemporalHelper` for all time-based condition checks — never raw timestamp comparisons
- Subscribers use `TickGuard` for idempotency — never custom implementations
- Heartbeat never calls any feature service — it only emits ticks
- `getLastTickTimestamp()` returns null on first launch — callers must handle null
- Adding a new time-based behavior requires zero changes to Heartbeat