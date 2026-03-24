**File Name**: feature_dependency_chain **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** the authoritative reference for feature ordering and allowed dependencies. Check this before adding any new cross-feature call, event subscription, or public function.

---

## The Rule

Dependencies flow downward only. A feature may call services or subscribe to events from features below it. It may never depend on a feature above it. Any upward arrow is a design violation.

**The test:** draw an arrow from the depending feature to the feature it depends on. All arrows must point downward.

---

## Feature Chain

```
Infrastructure      (foundation — no feature dependencies)
       ↓
Commitment          (CommitmentDefinition, CommitmentInstance)
       ↓
Activity            (LogEntry)
       ↓
Performance         (livePerformance, scores)
       ↓
┌─────────────────────────────────┐
│  Garment  Rewards  Encouragement│  (same level — no dependency between them)
└─────────────────────────────────┘
       ↓
Progression
       ↓
Analytics
       ↓
AI Insights
```

**User and Settings** — cross-cutting. Read by any feature needing user preferences or tier data. No downstream dependencies.

**Grace and NotificationScheduler** — infrastructure-level. Serve all features, depend only on Infrastructure and Commitment (for instance reads). NotificationSchedulerService subscribes to instance events and gates all notifications through `_shouldNotify(instance)` — only active commitments receive notifications.

**Config** — compile-time constants. Read by any feature. No dependencies.

---

## Key Design Principle

The public interface of Commitment is its instance events — not CommitmentEvent. CommitmentEvent is internal to the Commitment feature, consumed only by CommitmentIdentityService. No feature outside Commitment ever subscribes to it.

This means the entire application above Commitment has a uniform interface: subscribe to instance events, read instance fields. No feature needs to understand commitment lifecycle internals.

---

## Feature Interfaces

---

### Infrastructure

**Publishes:** LongIntervalTickEvent, ShortIntervalTickEvent, DayEndedEvent, WeekEndedEvent, WeekStartedEvent **Public functions:** EventBus.publish(), EventBus.on(), TemporalHelper, TickGuard

---

### Commitment

**Publishes (public):** InstanceCreatedEvent, InstanceUpdatedEvent, InstancePermanentlyDeletedEvent **Publishes (internal only):** CommitmentEvent — consumed only by CommitmentIdentityService **Public functions (CommitmentService):** getDefinition(), watchActiveCommitments(), watchFrozenCommitments(), watchDeletedCommitments(), watchCompletedCommitments(), getStateTransitionLog(), getPortfolioSize(), getActiveCount(), getRecentlyCreated() **Public functions (CommitmentIdentityService):** getInstances(from, to, definitionId?), watchInstancesForDay(), getCurrentInstance(), getInstanceForCommitmentOnDate(), getInstancesForDay(), getInstancesForWeek(), updateLivePerformance()

CommitmentService has no cross-feature dependencies — it publishes events and other features react. Access enforcement lives in UserCapabilityService (User feature), which reads from Commitment and Performance as valid downward calls.

`updateLivePerformance()` is the only write function exposed to features above. Performance calls it directly after calculating — a valid downward call.

---

### Activity

**Publishes:** ActivityEvent (created | updated | deleted) **Public functions:** recordEntry(), editEntry(), deleteEntry(), getTotalLoggedForCommitmentOnDate(), getEntriesForCommitment(), getEntriesForDay() **Subscribes to:** InstancePermanentlyDeletedEvent (Commitment) — for own data cleanup only **Calls directly:** nothing — fully independent except for the permanent delete subscription

---

### Performance

**Publishes:** PerformanceUpdatedEvent **Public functions:** getPerformanceForPeriod(), getDayScore(), getCommitmentWeekScore(), getOverallWeekScore() **Subscribes to:** InstanceCreatedEvent (Commitment), InstanceUpdatedEvent (Commitment), ActivityRecordedEvent (Activity) **Calls directly:** CommitmentIdentityService.updateLivePerformance(), CommitmentIdentityService.getInstanceForCommitmentOnDate(), CommitmentIdentityService.getInstances(), ActivityService.getTotalLoggedForCommitmentOnDate()

---

### Garment

**Subscribes to:** PerformanceUpdatedEvent **Calls directly:** CommitmentService.getDefinition(), PerformanceService.getPerformanceForPeriod()

---

### Rewards

**Subscribes to:** PerformanceUpdatedEvent (isClosed: true), WeekEndedEvent **Calls directly:** PerformanceService.getOverallWeekScore(), CommitmentIdentityService.getInstances()

---

### Encouragement

**Subscribes to:** ActivityRecordedEvent, PerformanceUpdatedEvent (isClosed: true), CupEarnedEvent, WeekEndedEvent **Calls directly:** PerformanceService.getDayScore()

---

### Progression

**Subscribes to:** CupEarnedEvent **Calls directly:** RewardService (cup history reads)

---

### Analytics

**Calls directly:** ActivityService.getEntriesForCommitment(), PerformanceService.getPerformanceForPeriod(), CommitmentIdentityService.getInstances()

---

### AI Insights

**Calls directly:** AnalyticsService (all reads)

---

### User / Settings

Cross-cutting — read by any feature needing user preferences or tier data. The User feature also owns user capability logic.

**Public functions (UserService):** getProfile(), getTier(), getPreferences() **Public functions (UserCapabilityService):** getBlockedReason(), canAddCommitment (Riverpod provider)

UserCapabilityService sits in the User feature and reads downward from Commitment and Performance to determine what the user is currently allowed to do. This is the correct placement — capability rules are a user concern, and the User feature sits high enough in the chain to read from all relevant features without any upward dependency.

---

## Violation Examples

|Violation|Why wrong|
|---|---|
|Any feature outside Commitment subscribes to CommitmentEvent|CommitmentEvent is internal|
|CommitmentIdentityService subscribes to PerformanceUpdatedEvent|Commitment watching Performance — upward|
|ActivityService calls PerformanceService.getDayScore()|Activity calling Performance — upward|
|PerformanceService subscribes to CupEarnedEvent|Performance watching Rewards — upward|
|CommitmentService calls RewardService|Commitment calling Rewards — upward|

---

## How to Use This Doc

**Before adding a subscription:** confirm the publishing feature is below the subscriber in the chain.

**Before adding a direct call:** confirm the called feature is below the caller in the chain.

**Before creating a new feature:** place it in the chain. Define its interface. Verify all dependencies point downward.

**When something feels wrong:** an upward dependency usually means the feature boundary is wrong. Redesign rather than accept the violation.