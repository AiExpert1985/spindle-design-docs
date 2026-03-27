**File Name**: architecture_rules **Phase**: All phases **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** rules every feature must follow. Enforces modularity, testability, and replaceability across all phases. Non-negotiable — every feature, every service, every repository must comply.

---

## 1. Layer Architecture

Four layers. Each layer communicates downward only — never upward, never sideways.

```
Presentation   →   Widgets, providers (passive observers only)
Application    →   Services, event streams (all business logic lives here)
Domain         →   Models, events (plain objects, no dependencies)
Data           →   Repositories, data sources (all external communication)
```

**Presentation** — UI only. Renders state. Forwards user intent to services. No calculations, no conditions representing business rules, no direct event subscriptions.

**Application** — all rules and logic live here. Services publish and subscribe to events. Services write state to providers for the presentation layer to observe. Features live in this layer, stacked in a dependency order — see §3.

**Domain** — plain model and event objects. No logic, no dependencies on other layers.

**Data** — storage and external APIs. Nothing above this layer knows what is behind it.

---

## 2. Feature Boundaries

The application is organized into features. Each feature has exactly one responsibility and owns its model, service, and repository. No feature reaches into another feature's internals.

**The service is the only public interface of a feature.** When one feature needs data from another, it calls that feature's service — never the repository, never the model directly. This is the _Interface Segregation_ principle applied at the feature level.

**A feature must be removable without changing any other feature.** The only impact of removal is that its events stop being published — subscribers stop receiving them and do nothing. If removing a feature requires editing another feature's code, the boundaries are wrong.

**Intra-feature calls are allowed.** Two services within the same feature may call each other directly. The boundary rules apply across features, not within them.

**The test:** can you describe what this feature does in one sentence? If not, it has too many responsibilities.

---

## 3. The Dependency Stack

Within the Application layer, features are stacked in a dependency order. A feature can only depend on features below it. Features at the same level cannot depend on each other. The full ordering is in `feature_dependency_chain`.

```
  [Higher features]      ← depend on everything below
        ↑
  [Mid-level features]   ← depend on lower features
        ↑
  [Lower features]       ← depend on nothing above them
```

Some features sit at the very base of the stack — used by many features above them, but depending on nothing themselves. They have no domain knowledge: no commitments, no instances, no users. They provide a raw capability and stop there.

**The test for a base feature:** can it be dropped into a completely different app with zero changes? If not, it contains domain-specific logic that does not belong at the base — move that logic upward into the appropriate feature, which then calls the base feature downward.

Think of this as a building. The foundation does not know what floors are built on top of it. A floor can use anything below it — it cannot use anything above it or beside it.

**No sideways communication.** If Feature A calls Feature B at the same level, it means A depends on B — so B must move below A. Two features at the same level that need to share data are a signal that responsibilities need to be re-examined.

**The test:** draw an arrow from the caller to the called. Does it point downward? If not, the dependency is forbidden.

---

## 4. Cross-Feature Communication

Three rules govern all communication across feature boundaries.

**Rule A — Read downward through services.** When a feature needs data owned by a lower feature, it calls that feature's service. Never access another feature's repository or model directly.

**Rule B — Notify upward through events.** When a feature's state changes, it publishes an event on its own stream. Any feature above it that needs to react subscribes to that stream. The publisher never calls the subscriber. The subscriber never polls the publisher.

This is the _Observer Pattern_. It is the only mechanism by which lower features communicate upward — they publish blindly, without knowing who (if anyone) is listening.

**Rule C — Never subscribe upward.** A feature only subscribes to events from features below it. A lower feature never reacts to events from a higher feature. If you find yourself subscribing upward, the logic belongs higher in the stack — move it, do not bend the rule.

**Events are for reactions, not queries.** If you need data, use Rule A. Events are only for broadcasting that something changed.

**Each feature owns its own event streams.** There is no centralized event bus. A feature exposes its events as part of its service interface. Upper features subscribe by importing the lower feature's service — which the stack already permits. This is intentional: a centralized bus would allow any feature to subscribe to any other feature's events regardless of stack position, making the one-way rule enforceable only by discipline. With per-feature streams, the rule is enforced structurally — a lower feature cannot import an upper feature, so it cannot subscribe to its events even if it wanted to.

---

## 5. Why Event-Driven

The problem with direct calls: if a lower feature needs to notify several higher features, it must import and call each of them. This creates upward dependencies — the foundation knows about the floors — which breaks the stack and creates circular dependencies.

Events solve this cleanly. The lower feature publishes to its own stream and stops. It never knows who reacts. Higher features subscribe independently, with no knowledge of each other.

**Each feature owns its own event streams — there is no centralized event bus.** A centralized bus would allow any feature to subscribe to any other feature's events regardless of stack position, making the one-way rule enforceable only by discipline. With per-feature streams, the rule is enforced structurally — a lower feature cannot import an upper feature, so it cannot subscribe to its events even if it wanted to.

**This design enables future expansion.** Adding a new feature that reacts to an existing event requires zero changes to anything already written. The lower feature keeps publishing. The new feature subscribes. Nothing else changes.

**Benefits:**

_No cycles._ A lower feature cannot accidentally call a higher feature — it can only publish to its own stream.

_Incremental addition._ New features subscribe to existing streams without touching existing code.

_Enforced separation._ Per-feature streams are a structural gap between publisher and subscriber — a feature cannot subscribe to events it has no right to see.

_Failure isolation._ A subscriber that throws does not affect the publisher or other subscribers.

_Isolated testing._ Emit a test event to verify a subscriber. Assert an event was emitted to verify a publisher.

**Drawbacks and mitigations:**

_Debugging is harder._ No linear call stack. Mitigation: every subscriber logs its activation. The event catalog in `event_catalog` lists every stream and its subscribers — keep it current.

_Execution order is not guaranteed._ If Subscriber B needs the result of Subscriber A's reaction, design A to publish a second event when done — B subscribes to that.

**Operating rules:**

- Publishers fire and forget — they never check who subscribed
- Subscribers are fully independent — they never know who published
- Events carry only the values that changed — no follow-up service call needed. Events never carry full model snapshots or unrelated data.
- A feature's internal events are not part of its public interface — only events consumed by features above it are public
- Events are never consumed from the presentation layer — UI observes providers
- Every subscription is cancelled when the service is disposed
- Use direct service calls for write-back operations, not events

---

## 6. Shared Low-Level Services

Some capabilities are needed by many features — time signals, time interpretation, notifications, error handling, logging. Rather than letting each feature implement its own version, each is built as a single feature with an abstract interface. Every feature that needs it calls the interface downward. The implementation behind the interface is swappable without touching any caller.

This is the _Separated Interface_ pattern: define the contract in one place, implement it elsewhere, callers depend only on the contract. It keeps features decoupled from platform details and makes future changes — a new notification backend, a remote logging service — a matter of swapping one implementation.

**Heartbeat** fires periodic time signals — a long-interval tick and a short-interval tick — carrying only a raw timestamp. Features that need time-based behavior subscribe to these streams. Heartbeat has no knowledge of what any subscriber will do with the timestamp. See `heartbeat`.

**TemporalHelper** answers semantic questions about time using the current user's preferences, and publishes day and week boundary events that any feature can subscribe to. It sits above UserCore and subscribes to Heartbeat. Callers pass only a timestamp to get an answer — no preferences handling required. Features that need to react to day or week boundaries subscribe to `TemporalHelperService` events rather than detecting boundaries independently. This centralizes all boundary detection in one place. See `service_temporal_helper`.

**Notifications** accept a payload from any feature and handle all delivery details. Features never call the OS notification API directly. The delivery mechanism — local notifications, push, in-app banners — can change without touching any feature. See `notification_service`.

**Error Handling** provides `Result<T>` as the standard return type for any function that can fail, and `AppError` as the structured failure type. Every service function that can fail returns `Result<T>` — no raw exceptions cross feature boundaries. See `error_handling`.

**Logger** is the single entry point for all diagnostic output — debug, info, warning, error. Nothing writes to the console or any storage directly. The Phase 1 implementation prints to console. Future implementations add persistent or remote output by swapping the implementation, with zero changes to any calling feature. See `logger`.

---

## 7. Presentation Layer

The presentation layer observes state — it never calculates, decides, or holds business logic.

**A widget has exactly two responsibilities:**

1. Render — transform current provider state into visible UI.
2. Forward intent — call a service function in direct response to a user action.

**A widget never:**

- Checks conditions to decide what to show — that belongs in a provider or service
- Calculates any derived value — that belongs in a service
- Contains logic representing a business rule
- Subscribes to event streams directly — it watches providers
- Calls multiple services to assemble data — a provider does that

**Providers are the assembly layer.** A provider may call multiple services to assemble the state a widget needs. No business logic — only data assembly. No widget writes to a provider directly.

**Providers must contain no logic.** A conditional, a calculation, or a decision in a provider belongs in a service. Create a dedicated service if no existing one fits. The symptom of drift: a provider that grows and starts making decisions.

---

## 8. Service Rules

- Pure logic only — no platform or UI framework imports
- Stateless where possible — same inputs, same outputs
- No side effects outside the service's own domain
- Services call other feature services only downward in the dependency chain
- Services within the same feature may call each other directly
- Services subscribe to events only from features below them in the chain
- All constants and thresholds come from configuration — never hardcoded
- All functions that can fail return `Result<T>` — no raw exceptions

---

## 9. Repository Rules

- A repository is called only by services within its own feature — no exceptions
- One abstract interface per feature, with multiple concrete implementations possible
- No business logic inside repositories — only fetch and store
- All IDs are client-generated — stable across storage backends
- Every model has `createdAt` and `updatedAt` timestamps
- Large collections always fetched with a time window or limit — never fetch all

Not every feature needs a repository. Features that only compute or produce ephemeral output need no storage.

---

## 10. Thin Model Principle

Every model contains only the fields that define what the object _is_ — its identity, its core state, and its timestamps. Fields that represent what other features _know about_ the object do not belong on the model.

When each feature owns its own data — linked by the object's ID — changes are completely contained within that feature.

**How to apply it:** when a feature needs to associate data with another feature's object, it creates its own model linked by that object's ID and subscribes to its events.

**The test:** if you removed this feature entirely, would the other feature's model need to change? If yes, the field does not belong on that model.

---

## 11. Type Safety

Code should make invalid states unrepresentable.

**Use enums for fixed value sets with no variant-specific data.** Any field with a known, bounded set of values must be an enum. Enums are validated at compile time, self-documenting, and refactor-safe.

Example — `CommitmentState`: every state is just a label, no state carries extra data. Enum is correct.

**Use sealed classes for variant types where some variants carry data.** A sealed class is an enum where one or more variants carry additional fields. The compiler enforces exhaustive handling.

Example — `Recurrence`: `Daily` and `Weekly` carry no extra data. `SpecificWeekDays` carries `weekDays`. A plain enum plus a nullable `weekDays` field creates a conditionally valid field. The sealed class eliminates this — `SpecificWeekDays` always has `weekDays`, others never do.

**Use classes for structured values.** When a concept has multiple related fields, model it as a named class. Maps are extended by convention and hope. Classes are extended by adding typed fields.

**Avoid primitive obsession.** A concept with two fields today will likely gain more. A class accommodates this without touching its consumers.

**The test:** if you deleted the type definition, would the compiler catch every misuse? If not, a class, enum, or sealed class is missing.

---

## 12. Configuration

All tunable constants live in a central configuration object. All user preference defaults live in a separate preferences defaults object.

- No service hardcodes a threshold, limit, interval, duration, or point value
- No service hardcodes a user preference default
- User preference defaults are applied once at account creation — never re-applied

---

## 13. Testability

- Every service testable by injecting a mock repository and mock event streams
- Emitting a test event verifies subscriber behavior without wiring up the publisher
- Asserting an event was emitted verifies publisher behavior without wiring up subscribers
- Widgets tested by providing mock providers — never by mocking services directly
- No test should require a running device, real database, or network connection
- Test priority: services first, repositories second, event handlers third, widgets last

---

## 14. Modularity Checklist

Before building any new feature:

- Does it have exactly one responsibility, describable in one sentence?
- Does it have its own model, service, and repository — or is there a clear reason one is not needed?
- Does it read all constants from configuration?
- Does it communicate with other features only through their service interfaces?
- Does it react to other features only through event subscriptions?
- Does it publish events when its state changes?
- Can it be removed without changing any other feature's code?