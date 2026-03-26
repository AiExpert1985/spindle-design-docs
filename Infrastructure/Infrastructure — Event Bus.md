**File Name**: infrastructure_eventbus **Feature**: Infrastructure **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** the in-process event bus. Provides the mechanism for decoupled communication between features. Publishers fire and forget. Subscribers react independently. Neither side knows the other exists.

For the full list of events and their payloads, see `event_catalog`. For who subscribes to what, see `feature_dependency_chain`.

---

## Implementation

```dart
abstract class AppEvent {}

class EventBus {
  final _controller = StreamController<AppEvent>.broadcast();
  void publish(AppEvent event) => _controller.add(event);
  Stream<T> on<T extends AppEvent>() => _controller.stream.whereType<T>();
  void dispose() => _controller.close();
}
```

One app-wide singleton, injected via dependency injection.

---

## Rules

- Never use the event bus from the presentation layer — UI observes Riverpod providers
- Events are immutable value objects — no logic, no mutable state
- Tick events carry timestamp only — subscribers interpret via TemporalHelper
- Publishers fire and forget — never check who subscribed
- Every subscription cancelled on service dispose
- Change events carry the values that changed — subscribers do not need a follow-up service call
- A feature's internal events are not part of its public interface — only events consumed by features above it are public
- Use direct service calls for write-back operations, not events