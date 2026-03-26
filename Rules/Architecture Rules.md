**File Name**: architecture_rules **Phase**: All phases **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** rules every feature must follow. Enforces modularity, testability, and replaceability across all phases. Non-negotiable — every feature, every service, every repository must comply.

---

## 1. Layer Architecture

Four layers. Each layer communicates downward only — never upward, never sideways.

```
Presentation   →   Widgets, providers (passive observers only)
Application    →   Services, event subscriptions (all business logic lives here)
Domain         →   Models, events (plain objects, no dependencies)
Infrastructure →   Truly feature-agnostic utilities: EventBus, TickService,
                   TemporalHelper, TickGuard, ErrorService
Data           →   Repositories, data sources (all external communication)
```

Within the Application layer, features are ordered in a strict dependency chain. See `feature_dependency_chain` for the full ordering. The chain runs from `UserCore` (lowest feature) through `Commitment`, `Activity`, `Performance`, the mid-level features, and up to `UserSettings` (highest feature). `Notification` sits just above `UserCore`, below `Commitment` — any feature may call it downward.

**Presentation** — UI only. Renders state. Forwards user intent to services. No calculations, no conditions representing business rules, no event bus access.

**Application** — all rules and logic live here. Services publish and subscribe to events. Services write state to providers for the presentation layer to observe.

**Domain** — plain model and event objects. No logic, no dependencies on other layers.

**Infrastructure** — cross-cutting utilities (clock, event bus, notifications, logging). No business logic, no domain models.

**Data** — storage and external APIs. Nothing above this layer knows what is behind it.

---

## 2. Feature Boundaries

The application is organized into features. Each feature has exactly one responsibility and owns its model, service, and repository. No feature reaches into another feature's internals.

**The service is the only public interface of a feature.** When one feature needs data from another, it calls that feature's service. It never touches the other feature's repository, model, or any internal detail.

**A feature must be removable without changing any other feature.** The only impact of removal is that its events stop being published — subscribers stop receiving them and do nothing. If removing a feature requires editing another feature's code, the boundaries are wrong.

**Intra-feature calls are allowed.** Two services within the same feature may call each other directly. The boundary rule applies across features, not within them.

**The test:** can you describe what this feature does in one sentence? If not, it has too many responsibilities.

---

## 3. Cross-Feature Communication

Three rules govern all communication across feature boundaries.

**Rule A — Read through services.** When a feature needs data owned by another feature, it calls that feature's service. Never access another feature's repository or model directly.

**Rule B — React through events.** When a feature's state changes and another feature needs to react, the first feature publishes an event. The second feature subscribes. The publisher never calls the subscriber directly. The subscriber never polls the publisher.

Events are for reactions to state changes, not for queries. A simple read does not need an event — Rule A handles that.

**Rule C — Dependencies flow one way.** Feature dependencies follow a fixed chain. No feature calls a feature above it in the chain. No circular dependencies. Ever. The chain is defined in `feature_dependency_chain`.

This rule has a corollary that is equally non-negotiable:

**Lower features never know upper features exist.** A lower feature publishes events and exposes service functions. It has no knowledge of who subscribes or who calls it. It never imports, calls, or subscribes to anything above it in the chain. Upper features watch lower features — never the reverse.

This is not a preference — it is the structural guarantee that makes the entire system maintainable. If a lower feature must know about an upper feature to do its work, the responsibilities are in the wrong place. Move the logic upward, not the dependency downward.

This principle corresponds to the Dependency Inversion Principle (SOLID) and is the core mechanism of Clean Architecture (Robert C. Martin). You do not need to know those names — the rule is self-evident from first principles: a foundation must not know what is built on top of it.

---

## 4. Event Bus

The event bus is the backbone of cross-feature communication.

**Why event-driven?**

The alternative is direct calls: Feature A changes state and calls B, C, and D in sequence. This creates tight coupling — A must know about every dependent, adding a feature requires editing A, a failure in D breaks A. Event-driven inverts this: A publishes one event and stops. Subscribers are added, removed, or changed without touching A.

**Benefits:**

_Incremental addition._ New features subscribe to existing events without modifying anything that already works.

_Enforced separation._ The event bus is a physical gap between publisher and subscriber.

_Isolated testing._ Publish a test event to verify a subscriber. Assert an event was published to verify a publisher.

_Failure isolation._ A subscriber that throws does not affect the publisher or other subscribers.

**Drawbacks and mitigations:**

_Debugging is harder._ No linear call stack. Mitigation: every subscriber logs its activation. The event catalog in `infrastructure_eventbus` lists every event and subscriber — keep it current.

_Execution order is not guaranteed._ If Subscriber B needs the result of Subscriber A's reaction, design A to publish a second event when done — B subscribes to that.

**Operating rules:**

- Publishers fire and forget — they never check who subscribed
- Subscribers are fully independent — they never know who published
- Change events carry the values that changed — no follow-up service call needed. Events never carry full model snapshots or unrelated data.
- A feature's internal events are not part of its public interface — only events consumed by features above it in the chain are public. Example: CommitmentEvent is internal to Commitment and never subscribed to outside it.
- The event bus is never used from the presentation layer — UI observes providers
- Every subscription is cancelled when the service is disposed
- Use direct service calls for write-back operations, not events

---

## 5. Presentation Layer

The presentation layer observes state — it never calculates, decides, or holds business logic.

**A widget has exactly two responsibilities:**

1. Render — transform current provider state into visible UI.
2. Forward intent — call a service function in direct response to a user action.

**A widget never:**

- Checks conditions to decide what to show — that lives in a provider or service
- Calculates any derived value — that lives in a service
- Contains logic representing a business rule
- Listens to events directly — it watches providers
- Calls multiple services to assemble data — a provider does that

**Providers are the assembly layer.** A provider may call multiple services to assemble the state a widget needs. No business logic — only data assembly. No widget writes to a provider directly.

**Providers must contain no logic.** A conditional, a calculation, or a decision in a provider belongs in a service. Create a dedicated service if no existing one fits. The symptom of drift: a provider that grows and starts making decisions.

---

## 6. Service Rules

- Pure logic only — no platform or UI framework imports
- Stateless where possible — same inputs, same outputs
- No side effects outside the service's own domain
- Services call other feature services only downward in the dependency chain
- Services within the same feature may call each other directly
- Services subscribe to events only from features they depend on — never from features above them
- All constants and thresholds come from configuration — never hardcoded

---

## 7. Repository Rules

- A repository is called only by services within its own feature — no exceptions
- One abstract interface per feature, with multiple concrete implementations possible
- No business logic inside repositories — only fetch and store
- All IDs are client-generated — stable across storage backends
- Every model has `createdAt` and `updatedAt` timestamps
- Large collections always fetched with a time window or limit — never fetch all

Not every feature needs a repository. Features that only compute or produce ephemeral output need no storage.

---

## 8. Thin Model Principle

Every model contains only the fields that define what the object _is_ — its identity, its core state, and its timestamps. Fields that represent what other features _know about_ the object do not belong on the model.

When each feature owns its own data — linked by the object's ID — changes are completely contained within that feature.

**How to apply it:** when an external feature needs to associate data with an object, it creates its own model linked by the object's ID and subscribes to the object's events.

**The test:** if you removed this feature entirely, would the object's model need to change? If yes, the field does not belong on the object.

---

## 9. Type Safety

Code should make invalid states unrepresentable.

**Use enums for fixed value sets with no variant-specific data.** Any field with a known, bounded set of values must be an enum. Enums are validated at compile time, self-documenting, and refactor-safe.

Example — `CommitmentState`: every state is just a label, no state needs extra data. Enum is correct.

**Use sealed classes for variant types where some variants carry data.** A sealed class is an enum where one or more variants carry additional fields. The compiler enforces exhaustive handling.

Example — `Recurrence`: `Daily` and `Weekly` carry no extra data. `SpecificWeekDays` carries `weekDays`. The alternative — a plain enum plus a nullable `weekDays` field elsewhere — creates a conditionally valid field. The sealed class eliminates this: `SpecificWeekDays` always has `weekDays`, others never do.

Sealed classes are also correct for events that carry variant-specific data.

**Use classes for structured values.** When a concept has multiple related fields, model it as a named class. Classes provide type safety and are extended by adding typed fields. Maps are extended by convention and hope.

**Avoid primitive obsession.** A concept with two fields today will likely gain more. A class accommodates this without touching its consumers.

**The test:** if you deleted the type definition, would the compiler catch every misuse? If not, a class, enum, or sealed class is missing.

---

## 10. Configuration

All tunable constants live in a central configuration object. All user preference defaults live in a separate preferences defaults object.

- No service hardcodes a threshold, limit, interval, duration, or point value
- No service hardcodes a user preference default
- User preference defaults are applied once at account creation — never re-applied

---

## 11. Infrastructure Utilities

Infrastructure utilities are available to all features. No business logic, no domain models.

**Infrastructure is the foundation. It has no knowledge of any feature.**

Infrastructure never imports, subscribes to, or calls any feature service, feature event, or feature model. It provides capabilities — clocks, event delivery, notification dispatch, logging — and stops there. Features use infrastructure; infrastructure never uses features.

This is not a matter of preference. Infrastructure that knows about features is no longer infrastructure — it is a feature with extra responsibilities. The moment infrastructure subscribes to an `InstanceCreatedEvent`, it has taken on scheduling logic that belongs in a feature service. That logic must be moved up into the appropriate feature, which then calls infrastructure downward.

**The test:** can this infrastructure utility be dropped into a completely different app with no changes? If not, it contains feature-specific logic that does not belong here.

**Event bus** — delivers events between features. Knows nothing about what the events contain or who cares about them.

**Tick service** — fires long-interval and short-interval ticks. Publishes raw timestamps only. No knowledge of what any subscriber will do with them.

**Temporal helper** — answers semantic time questions using user preferences. No knowledge of commitments, instances, or any domain concept.

**Tick guard** — shared idempotency for tick subscribers. Every tick subscriber uses this.

**Notification service** — delivers a notification payload to the OS. Knows nothing about what triggered it or what feature it concerns.

**Notification tracking** — records that a notification was sent, keyed by an arbitrary ID. No knowledge of what the ID represents.

**Grace service** — schedules a follow-up callback after a configurable delay. No knowledge of what grace means in the domain.

**Logging service** — records diagnostic output. No knowledge of what generated it.

**Error handler** — centralised handling for unhandled exceptions. No feature implements its own top-level error handling.

---

## 12. Testability

- Every service testable by injecting a mock repository and a mock event bus
- Publishing a test event verifies subscriber behavior without wiring up the publisher
- Subscribing to a mock event bus verifies publisher behavior without wiring up subscribers
- Widgets tested by providing mock providers — never by mocking services directly
- No test should require a running device, real database, or network connection
- Test priority: services first, repositories second, event handlers third, widgets last

---

## 13. Modularity Checklist

Before building any new feature:

- Does it have exactly one responsibility, describable in one sentence?
- Does it have its own model, service, and repository — or is there a clear reason one is not needed?
- Does it read all constants from configuration?
- Does it communicate with other features only through their service interfaces?
- Does it react to other features only through event subscriptions?
- Does it publish events when its state changes?
- Can it be removed without changing any other feature's code?

---

## 14. Core Feature

`core` is for screens and components that assemble data from multiple features and cannot be attributed to any single one. Presentation only.

**Belongs in `core`:** screens that draw from three or more features to give a unified view. The app-level navigation shell.

**Does not belong in `core`:** infrastructure utilities, configuration, or anything that can be cleanly attributed to one feature.

**Hard rules:** no services, no repositories, no models. No business logic. Reads other features only through their public service interfaces.

**The test:** can you name one feature that owns this screen? If yes — put it there. If no — it belongs in `core`.