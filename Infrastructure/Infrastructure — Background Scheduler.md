**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Phase**: 1 **Feature**: Infrastructure

**Purpose:** a clock with two interval tiers. Publishes `LongIntervalTickEvent` and `ShortIntervalTickEvent` carrying only a raw timestamp. Tick events carry no pre-computed booleans — subscribers use `TemporalHelper` to interpret what a timestamp means for their user. The scheduler knows nothing about features, boundaries, or preferences.

---

## Design Philosophy

The scheduler applies the same event bus philosophy to time. It announces that time has passed and steps back. What that moment means — is it midnight? is it the week boundary? is it waking hours? — is determined by `TemporalHelper` inside each subscriber using the user's own preferences.

Three things are strictly separated:

```
Scheduler      → fires at intervals, publishes raw timestamps
TemporalHelper → interprets timestamps using user preferences
Subscribers    → decide what to do when conditions are met
```

None of these knows about the others' concerns.

---

## Two-Layer Design

```
Subscribers (feature services using TemporalHelper)
    ↑  LongIntervalTickEvent / ShortIntervalTickEvent
TickService (abstract)         ← the abstraction
    ↓
TickServiceImpl                ← WorkManager implementation (swappable)
    ↓
BackgroundSchedulerAdapter     ← WorkManager/BGTaskScheduler abstraction
```

---

## TickService Abstraction

```dart
abstract class TickService {
  // Long interval — ~15 minutes. For things that can wait.
  // Instance generation, week boundaries, cup calculation triggers,
  // day celebration timing, AI summary triggers.
  Stream<LongIntervalTickEvent> get longIntervalTick;

  // Short interval — ~1 minute. For things where users notice delay.
  // Window warnings, predictive friction, realtime status checks.
  Stream<ShortIntervalTickEvent> get shortIntervalTick;
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

No booleans. No pre-computed fields. Raw timestamp only. `TemporalHelper` does the interpretation.

---

## Phase 1 Implementation

One WorkManager periodic task, ~15 minutes. Both streams receive the same tick.

```dart
class DefaultTickService implements TickService {
  final _controller = StreamController<TickEvent>.broadcast();

  @override
  Stream<LongIntervalTickEvent> get longIntervalTick =>
      _controller.stream.map((e) => LongIntervalTickEvent(e.timestamp));

  @override
  Stream<ShortIntervalTickEvent> get shortIntervalTick =>
      _controller.stream.map((e) => ShortIntervalTickEvent(e.timestamp));
  // Same underlying stream — short interval subscribers get ~15 min ticks in Phase 1

  void onWorkManagerFire(DateTime now) {
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

  @override
  Stream<LongIntervalTickEvent>  get longIntervalTick  => _long.stream;
  @override
  Stream<ShortIntervalTickEvent> get shortIntervalTick => _short.stream;

  void onLongFire(DateTime now)  => _long.add(LongIntervalTickEvent(now));
  void onShortFire(DateTime now) => _short.add(ShortIntervalTickEvent(now));
}
```

Swapping from `DefaultTickService` to `DualTickService` requires changing one dependency injection binding. Every subscriber is completely unaffected — they subscribed to `longIntervalTick` or `shortIntervalTick`, not to an implementation.

---

## Who Subscribes to Which

**Long interval tick:**

|Subscriber|Condition checked via TemporalHelper|Action|
|---|---|---|
|`CommitmentSchedulerService`|`isDayBoundary`|Generate instances, finalize windows|
|`CommitmentSchedulerService`|`isWeekBoundary`|Publish `WeekEndedEvent`, `WeekStartedEvent`|
|`EncouragementService`|near `celebrationTime` preference|Evaluate day celebration Path B|
|`WeeklySummaryService`|`isWeekBoundary`|Trigger AI quick summary (Phase 3)|

**Short interval tick:**

|Subscriber|Condition checked via TemporalHelper|Action|
|---|---|---|
|`CommitmentSchedulerService`|`isWakingHours`|Window warnings, grace expiry checks|
|`PredictiveFrictionService`|`isWakingHours`|Risk signal evaluation|

---

## App Open Catchup

If ticks were missed while the app was closed:

1. Read `lastTickTimestamp` from local storage
2. For each missed interval slot, publish `LongIntervalTickEvent` with the historical timestamp
3. Subscribers process catchup ticks through their normal `TickGuard` — idempotency prevents double work

Short interval catchup is not replayed — only long interval matters for catchup. Missed short-interval checks (window warnings) are harmless to replay at the next real tick.

---

## Platform Accuracy

WorkManager does not guarantee exact intervals. Long interval: expect 10–20 minutes in practice. Short interval: expect 1–3 minutes in practice. This is acceptable — `TemporalHelper` conditions are checked on each tick arrival regardless of exact timing.

---

## Rules

- `TickService` is the abstraction — never reference `DefaultTickService` or `DualTickService` outside dependency injection
- Tick events carry timestamp only — no pre-computed booleans
- Subscribers use `TemporalHelper` for all time-based condition checks — never raw timestamp comparisons
- Subscribers use `TickGuard` for idempotency — never custom implementations
- The scheduler never calls any feature service — it only fires ticks
- Adding a new time-based behavior requires zero changes to the scheduler