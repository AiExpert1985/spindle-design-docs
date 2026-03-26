**File Name**: feature_dependency_chain **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the authoritative reference for feature ordering and allowed dependencies. Check this before adding any new cross-feature call, event subscription, or public function.

---

## The Rule

Dependencies flow downward only. A feature may call services or subscribe to events from features below it. It may never depend on a feature above it. Any upward arrow is a design violation.

**The test:** draw an arrow from the depending feature to the feature it depends on. All arrows must point downward.

---

## Feature Chain

```
Infrastructure        (EventBus, TemporalHelper, Achievable interface,
                       AchievementRecord, AchievementType)
      ↓
Commitment            ✓ LOCKED
      ↓
Activity              ✓ LOCKED
      ↓
Performance           ✓ LOCKED
      ↓
┌────────────────────────────────────────────────────┐
│  Streak · Cups · Rewards · Analytics               │  same level — each independent
│  Streak publishes StreakChangedEvent                │  no dependency between them
└────────────────────────────────────────────────────┘
      ↓
┌────────────────────────────────────────────────────┐
│  Milestones                                        │  depends on Streak only
│  subscribes to StreakChangedEvent                  │
└────────────────────────────────────────────────────┘
      ↓
┌────────────────────────────────────────────────────┐
│  Garment · Achievements                            │  same level — no dependency
│  Garment reads Streak                              │  between them
│  Achievements subscribes to Cup, Reward, Milestone │
└────────────────────────────────────────────────────┘
      ↓
Progression           (ScoringService · ProgressionService — internal)
      ↓
Encouragement
      ↓
┌────────────────────────────────────────────────────┐
│  AI Insights · Core                                │  same level — no dependency
└────────────────────────────────────────────────────┘
```

---

## Key Design Principles

**Commitment's public interface is instance events.** `CommitmentEvent` is internal. No feature outside Commitment subscribes to it.

**Performance is the score authority.** No feature above Performance calculates scores independently.

**`Achievable` is the achievement contract.** Any feature whose domain model implements `Achievable` participates in the achievement system. `AchievementService` subscribes to their events and calls `toAchievementRecord()` — it never imports feature internals.

**Achievements is a pure aggregator.** It owns `AchievementRecord` storage and the unified read interface. It does not own the logic of what constitutes an achievement — that belongs to each producing feature.

**Progression never knows what an achievement is.** `ScoringService` translates `AchievementEarnedEvent` to points. `ProgressionService` only knows about points and levels.

**Same-level features are fully independent.** No feature calls or subscribes to a feature at the same level.

---

## Locked Features

**Commitment** — `CommitmentDefinition`, `CommitmentInstance`, `CommitmentService`, `CommitmentIdentityService`. Public interface: instance events and read functions below.

**Activity** — `LogEntry`, `ActivityService`, `ActivityRepository`. Public interface: `ActivityEvent` and read/write functions below.

**Performance** — `PerformanceService`. Public interface: `PerformanceUpdatedEvent` and query functions below.

---

## Feature Interfaces

---

### Infrastructure

**Publishes:** LongIntervalTickEvent, ShortIntervalTickEvent, DayEndedEvent, WeekEndedEvent, WeekStartedEvent

**Shared contracts:** Achievable (interface), AchievementRecord (model), AchievementType (enum)

**Public functions:** EventBus.publish(), EventBus.on(), TemporalHelper, TickGuard

---

### Commitment ✓ LOCKED

**Publishes (public):** InstanceCreatedEvent, InstanceUpdatedEvent, InstancePermanentlyDeletedEvent

**Publishes (internal only):** CommitmentEvent — consumed only by CommitmentIdentityService

**Public functions (CommitmentService):** getDefinition(), watchActiveCommitments(), watchFrozenCommitments(), watchDeletedCommitments(), watchCompletedCommitments(), getStateTransitionLog(), getPortfolioSize(), getActiveCount(), getRecentlyCreated()

**Public functions (CommitmentIdentityService):** getInstances(), watchInstancesForDay(), getCurrentInstance(), getInstanceForCommitmentOnDate(), getInstancesForDay(), getInstancesForWeek(), updateLivePerformance()

---

### Activity ✓ LOCKED

**Publishes:** ActivityEvent (created | updated | deleted)

**Public functions:** recordEntry(), editEntry(), deleteEntry(), getTotalLoggedForCommitmentOnDate(), getEntriesForCommitment(), getEntriesForDay()

**Subscribes to:** InstancePermanentlyDeletedEvent (Commitment)

---

### Performance ✓ LOCKED

**Publishes:** PerformanceUpdatedEvent(instanceId, definitionId, windowStart, livePerformance, isClosed)

**Public functions:** getPerformanceForPeriod(), getDayScore(), getCommitmentWeekScore(), getOverallWeekScore(), isWindowSuccess(livePerformance)

**Subscribes to:** InstanceCreatedEvent, InstanceUpdatedEvent (Commitment), ActivityEvent (Activity)

**Calls directly:** CommitmentIdentityService.updateLivePerformance(), CommitmentIdentityService.getInstanceForCommitmentOnDate(), CommitmentIdentityService.getInstances(), ActivityService.getTotalLoggedForCommitmentOnDate()

---

### Streak

**Implements Achievable on:** `StreakRecord` does NOT implement Achievable — streaks are continuous state. `MilestoneRecord` (owned by Achievements-internal MilestoneService) implements Achievable for milestone moments.

**Publishes:** StreakChangedEvent(definitionId, currentStreak, bestStreak)

**Public functions:** getStreakRecord(definitionId), watchStreakRecord(definitionId), getBestStreakOverall()

**Subscribes to:** PerformanceUpdatedEvent (Performance), InstancePermanentlyDeletedEvent (Commitment)

**Calls directly:** PerformanceService.isWindowSuccess()

---

### Cups

**Implements Achievable on:** `WeeklyCup`

**Publishes:** CupEarnedEvent(cup: WeeklyCup)

**Public functions:** getAllCups(), getCupsSince(from)

**Subscribes to:** WeekEndedEvent (Infrastructure)

**Calls directly:** PerformanceService.getOverallWeekScore()

---

### Rewards

**Implements Achievable on:** `RewardRecord`

**Publishes:** RewardEarnedEvent(record: RewardRecord)

**Public functions:** getLatestReward()

**Subscribes to:** WeekEndedEvent (Infrastructure)

**Calls directly:** PerformanceService.getPerformanceForPeriod()

---

### Analytics

**Public functions:** computeCommitmentFacts(), computeWeeklyFacts(), computeDayFacts()

**Calls directly:** ActivityService, PerformanceService, CommitmentIdentityService, StreakService.getStreakRecord() (when streak history needed)

No events subscribed. No events published. Pure computation on demand.

---

### Milestones

**Implements Achievable on:** `MilestoneRecord`

**Publishes:** MilestoneEarnedEvent(record: MilestoneRecord)

**Subscribes to:** StreakChangedEvent (Streak), InstancePermanentlyDeletedEvent (Commitment)

**Calls directly:** CommitmentService.getDefinition() — name snapshot only

---

### Garment

**Publishes:** GarmentUpdatedEvent(definitionId, completionPercent, weeklyDelta)

**Public functions:** watchGarmentProfile(), getWeeklyProgress(), getLiveWeekDelta()

**Subscribes to:** InstanceCreatedEvent (Commitment), PerformanceUpdatedEvent where isClosed: true (Performance), InstancePermanentlyDeletedEvent (Commitment), WeekEndedEvent (Infrastructure)

**Calls directly:** CommitmentService.getDefinition(), StreakService.getStreakRecord() (via AcceleratorService)

---

### Achievements

**Publishes (public):** AchievementEarnedEvent(record: AchievementRecord)

**Public functions (AchievementService):** getAchievements(limit?, type?), watchRecentAchievements(), getCupHistory(), getCupsSince(from), getStreakRecord(definitionId), getBestStreakOverall()

**Subscribes to:** CupEarnedEvent (Cups), RewardEarnedEvent (Rewards), MilestoneEarnedEvent (Milestones), InstancePermanentlyDeletedEvent (Commitment)

**Calls directly:** each event's model `.toAchievementRecord()` via Achievable interface

---

### Progression

**Internal services:** ScoringService (AchievementEarnedEvent → PointsAwardedEvent), ProgressionService (PointsAwardedEvent → LevelReachedEvent)

**Publishes:** LevelReachedEvent, PointsAwardedEvent (internal)

**Public functions:** getProgressionSummary(), watchProgressionSummary(), getCurrentWeekProjection(), isReferralUnlocked()

**Subscribes to:** AchievementEarnedEvent (Achievements) — consumed by ScoringService

**Calls directly:** PerformanceService.getOverallWeekScore() — current week projection only

---

### Encouragement

**Subscribes to:** ActivityEvent (Activity), PerformanceUpdatedEvent where isClosed: true (Performance), AchievementEarnedEvent (Achievements), LevelReachedEvent (Progression), WeekEndedEvent (Infrastructure)

**Calls directly:** PerformanceService.getDayScore(), CommitmentIdentityService.getInstancesForDay(), AnalyticsService.computeDayFacts()

---

### AI Insights

**Public functions:** generateMicroInsight(), generateQuickSummary(), generateDeepReport()

**Calls directly:** AnalyticsService (all reads)

---

### Core

Presentation only. Screens and components that assemble data from multiple features. No services, no repositories, no models of their own.

---

### User / Settings

Cross-cutting — read by any feature needing user preferences or tier data.

**Public functions (UserService):** getProfile(), getTier(), getPreferences(), getTemporalPreferences(), update*() functions

**Public functions (UserCapabilityService):** getBlockedReason(), canAddCommitment (Riverpod provider)

---

## Violation Examples

|Violation|Why wrong|
|---|---|
|Any feature outside Commitment subscribes to CommitmentEvent|Internal event|
|Cups calls StreakService|Same-level dependency|
|Rewards calls CupService|Same-level dependency|
|Streak subscribes to CupEarnedEvent|Same-level dependency|
|Milestones subscribes to CupEarnedEvent|Milestones sits above Streak only — Cup is same-level|
|Milestones calls RewardService|Same-level dependency|
|Achievements calls ProgressionService|Upward dependency|
|Garment calls AchievementService|Same-level dependency — Garment reads Streak directly|
|ProgressionService reads AchievementService directly|ProgressionService only knows points — ScoringService handles achievements|
|Any feature implements Achievable on a service|Achievable belongs on domain models only|
|WeeklyCup.toAchievementRecord() calls a service|Must be pure — no side effects|

---

## How to Use This Doc

**Before adding a subscription:** confirm the publishing feature is below the subscriber.

**Before adding a direct call:** confirm the called feature is below the caller.

**Before creating a new achievement-producing feature:** implement `Achievable` on its domain model. Publish an event carrying that model. `AchievementService` subscribes automatically — no registration needed.

**Before modifying a locked feature:** treat as a breaking change. Review all consumers.