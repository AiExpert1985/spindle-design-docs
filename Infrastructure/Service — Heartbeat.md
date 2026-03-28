**File Name**: heartbeat **Feature**: Infrastructure **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** the app's pulse. Fires periodic signals — a long-interval tick and a short-interval tick — carrying only a raw timestamp. Any feature that needs time-based behavior subscribes to the appropriate stream. Heartbeat knows nothing about what the timestamp means, who subscribes, or what any subscriber will do.

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
}
```

Both events carry only a timestamp:

```dart
class LongIntervalTick  { final DateTime timestamp; }
class ShortIntervalTick { final DateTime timestamp; }
```

---

## App Open Catch-Up

No special catch-up logic is needed. Each subscriber processes everything due at or before the tick timestamp — so when the app opens and the first WorkManager tick fires, any work missed while the app was closed is handled naturally through the same path as a live tick. Subscribers must be written with this in mind: given a timestamp, do all work that was due at or before it.

---

## Phase 1 Implementation

One WorkManager periodic task, ~15 minutes. Both streams receive the same tick.

```dart
class DefaultTickService implements TickService {
  final _controller = StreamController<DateTime>.broadcast();

  @override
  Stream<LongIntervalTick> get longIntervalTick =>
      _controller.stream.map((t) => LongIntervalTick(t));

  @override
  Stream<ShortIntervalTick> get shortIntervalTick =>
      _controller.stream.map((t) => ShortIntervalTick(t));

  @override
  void onFire(DateTime now) => _controller.add(now);
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

  @override
  Stream<LongIntervalTick>  get longIntervalTick  => _long.stream;
  @override
  Stream<ShortIntervalTick> get shortIntervalTick => _short.stream;

  void onLongFire(DateTime now)  => _long.add(LongIntervalTick(now));
  void onShortFire(DateTime now) => _short.add(ShortIntervalTick(now));
}
```

Swapping from `DefaultTickService` to `DualTickService` requires changing one dependency injection binding. Every subscriber is unaffected.

---

## Platform Accuracy

WorkManager does not guarantee exact intervals. Long interval: expect 10–20 minutes in practice. Short interval: expect 1–3 minutes. This is acceptable for a commitment helper — no behavior is time-critical to the minute.

---

## Rules

- `TickService` is the abstraction — never reference `DefaultTickService` or `DualTickService` outside dependency injection
- Tick events carry timestamp only — no pre-computed booleans, no domain knowledge
- Subscribers process everything due at or before the tick timestamp — this is what makes catch-up free
- Heartbeat never calls any feature service — it only emits ticks
- Adding a new time-based behavior requires zero changes to Heartbeat