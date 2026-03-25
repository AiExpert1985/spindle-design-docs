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
Infrastructure
      ↓
Commitment                    (CommitmentDefinition, CommitmentInstance) ✓ LOCKED
      ↓
Activity                      (LogEntry) ✓ LOCKED
      ↓
Performance                   (livePerformance, scores) ✓ LOCKED
      ↓
Achievements                  (Streak · Cups · Milestones · Rewards — internal)
      ↓
┌──────────────────────────────────────┐
│  Garment · Analytics                 │  same level — no dependency between them
└──────────────────────────────────────┘
      ↓
Progression                   (ScoringService · ProgressionService — internal)
      ↓
Encouragement
      ↓
┌──────────────────────────────────────┐
│  AI Insights · Core                  │  same level — no dependency between them
└──────────────────────────────────────┘
```

**Dependency notes:**

- **Achievements** contains Streak, Cups, Milestones, and Rewards as internal services. Streak is internal — `StreakChangedEvent` is never published outside Achievements. The public interface is `AchievementService` and `AchievementEarnedEvent`.
- **Garment** depends on Performance and calls `AchievementService.getStreakRecord()` for the accelerator. It does not subscribe to any Achievements event — it reads streak data synchronously when a window closes.
- **Analytics** depends on Performance, Activity, and Commitment. It may call `AchievementService` for streak history when needed — a valid downward call.
- **Progression** contains `ScoringService` (converts achievements to points) and `ProgressionService` (converts points to levels) as internal layers. `ProgressionService` never reads from Achievements directly — only from `ScoringService`.
- **Encouragement** depends on both Achievements (`AchievementEarnedEvent`) and Progression (`LevelReachedEvent`).
- **AI Insights** and **Core** are at the same level — neither depends on the other.

**User / Settings** — cross-cutting. Read by any feature needing user preferences or tier data. No downstream dependencies.

**Grace and NotificationScheduler** — infrastructure-level. Serve all features, depend only on Infrastructure and Commitment for instance reads.

**Config** — compile-time constants. Read by any feature. No dependencies.

---

## Locked Features

Three features are locked — their models, services, and repositories are settled and can be built against with confidence.

**Commitment** — `CommitmentDefinition`, `CommitmentInstance`, `CommitmentService`, `CommitmentIdentityService`, `CommitmentRepository`, `CommitmentInstanceRepository`. The public interface is instance events and the read functions below.

**Activity** — `LogEntry`, `ActivityService`, `ActivityRepository`. The public interface is `ActivityEvent` and the read/write functions below.

**Performance** — `PerformanceService`. The public interface is `PerformanceUpdatedEvent` and the query functions below. No stored model — all scores calculated on demand from instances.

Features above the locked layer build on these three as a stable foundation. Changes to locked features require explicit design review.

---

## Key Design Principles

**Commitment's public interface is instance events.** `CommitmentEvent` is internal — consumed only by `CommitmentIdentityService`. No feature outside Commitment ever subscribes to it.

**Performance is the score authority.** No feature above Performance calculates scores independently. All performance data flows through `PerformanceService`.

**Achievements is the recognition authority.** All recognition of user behaviour — streaks, cups, milestones, rewards — lives inside Achievements. Internal services coordinate via internal events. The outside world sees only `AchievementService` and `AchievementEarnedEvent`.

**Progression never knows what an achievement is.** `ScoringService` translates achievements to points. `ProgressionService` only knows about points and levels. Adding a new achievement type requires no change to `ProgressionService`.

**Same-level features are fully independent.** No feature calls or subscribes to a feature at the same level.

---

## Feature Interfaces

---

### Infrastructure

**Publishes:** LongIntervalTickEvent, ShortIntervalTickEvent, DayEndedEvent, WeekEndedEvent, WeekStartedEvent

**Public functions:** EventBus.publish(), EventBus.on(), TemporalHelper, TickGuard

---

### Commitment ✓ LOCKED

**Publishes (public):** InstanceCreatedEvent, InstanceUpdatedEvent, InstancePermanentlyDeletedEvent

**Publishes (internal only):** CommitmentEvent — consumed only by CommitmentIdentityService

**Public functions (CommitmentService):** getDefinition(), watchActiveCommitments(), watchFrozenCommitments(), watchDeletedCommitments(), watchCompletedCommitments(), getStateTransitionLog(), getPortfolioSize(), getActiveCount(), getRecentlyCreated()

**Public functions (CommitmentIdentityService):** getInstances(from, to, definitionId?), watchInstancesForDay(), getCurrentInstance(), getInstanceForCommitmentOnDate(), getInstancesForDay(), getInstancesForWeek(), updateLivePerformance()

---

### Activity ✓ LOCKED

**Publishes:** ActivityEvent (created | updated | deleted)

**Public functions:** recordEntry(), editEntry(), deleteEntry(), getTotalLoggedForCommitmentOnDate(), getEntriesForCommitment(), getEntriesForDay()

**Subscribes to:** InstancePermanentlyDeletedEvent (Commitment) — own data cleanup only

---

### Performance ✓ LOCKED

**Publishes:** PerformanceUpdatedEvent(instanceId, definitionId, windowStart, livePerformance, isClosed)

**Public functions:** getPerformanceForPeriod(from, to, definitionId?), getDayScore(date), getCommitmentWeekScore(definitionId, weekStart), getOverallWeekScore(weekStart), isWindowSuccess(livePerformance)

**Subscribes to:** InstanceCreatedEvent, InstanceUpdatedEvent (Commitment), ActivityEvent (Activity)

**Calls directly:** CommitmentIdentityService.updateLivePerformance(), CommitmentIdentityService.getInstanceForCommitmentOnDate(), CommitmentIdentityService.getInstances(), ActivityService.getTotalLoggedForCommitmentOnDate()

---

### Achievements

**Internal services:** StreakService, CupService, MilestoneService, RewardService

**Internal events (never published outside Achievements):** StreakChangedEvent, CupEarnedInternalEvent, MilestoneReachedInternalEvent, RewardEarnedInternalEvent

**Publishes (public):** AchievementEarnedEvent(record: AchievementRecord)

**Public functions (AchievementService):** getAchievements(limit?, type?), watchRecentAchievements(), getCupHistory(), getStreakRecord(definitionId), getBestStreakOverall()

**Subscribes to:** PerformanceUpdatedEvent (Performance), WeekEndedEvent (Infrastructure), InstancePermanentlyDeletedEvent (Commitment)

**Calls directly:** PerformanceService.getOverallWeekScore(), PerformanceService.isWindowSuccess()

---

### Garment

**Publishes:** GarmentUpdatedEvent(definitionId, completionPercent, weeklyDelta)

**Public functions:** watchGarmentProfile(definitionId), getWeeklyProgress(definitionId, limit?), getLiveWeekDelta(definitionId)

**Subscribes to:** InstanceCreatedEvent (Commitment), PerformanceUpdatedEvent where isClosed: true (Performance), InstancePermanentlyDeletedEvent (Commitment), WeekEndedEvent (Infrastructure)

**Calls directly:** CommitmentService.getDefinition(), AchievementService.getStreakRecord(definitionId)

---

### Analytics

**Public functions:** computeCommitmentFacts(definitionId, from, to, includeNotes), computeWeeklyFacts(from, to, includeNotes), computeDayFacts(date)

**Calls directly:** ActivityService.getEntriesForCommitment(), ActivityService.getEntriesForDay(), PerformanceService.getPerformanceForPeriod(), PerformanceService.getDayScore(), CommitmentIdentityService.getInstances(), AchievementService.getStreakRecord() (when streak history needed for analysis)

No events subscribed. No events published. Pure computation on demand.

---

### Progression

**Internal services:** ScoringService (achievements → points), ProgressionService (points → levels)

**Publishes:** LevelReachedEvent(newLevel, levelName, levelUpMessage, previousLevel), PointsAwardedEvent(points, source, earnedAt)

**Public functions:** getProgressionSummary(), watchProgressionSummary(), getCurrentWeekProjection(), isReferralUnlocked()

**Subscribes to:** AchievementEarnedEvent (Achievements) — consumed by ScoringService

**Calls directly:** PerformanceService.getOverallWeekScore() — for current week projection only

---

### Encouragement

**Subscribes to:** ActivityEvent (Activity), PerformanceUpdatedEvent where isClosed: true (Performance), AchievementEarnedEvent (Achievements), LevelReachedEvent (Progression), WeekEndedEvent (Infrastructure)

**Calls directly:** PerformanceService.getDayScore(), AnalyticsService.computeDayFacts()

---

### AI Insights

**Public functions:** generateMicroInsight(), generateQuickSummary(), generateDeepReport()

**Calls directly:** AnalyticsService (all reads)

---

### Core

Presentation only. Screens and components that assemble data from three or more features. No services, no repositories, no models. See `architecture_rules` section 14.

---

### User / Settings

Cross-cutting — read by any feature needing user preferences or tier data.

**Public functions (UserService):** getProfile(), getTier(), getPreferences()

**Public functions (UserCapabilityService):** getBlockedReason(), canAddCommitment (Riverpod provider)

---

## Violation Examples

|Violation|Why wrong|
|---|---|
|Any feature outside Commitment subscribes to CommitmentEvent|CommitmentEvent is internal|
|Any feature outside Achievements subscribes to StreakChangedEvent|StreakChangedEvent is internal to Achievements|
|CommitmentIdentityService subscribes to PerformanceUpdatedEvent|Commitment watching Performance — upward|
|ActivityService calls PerformanceService.getDayScore()|Activity calling Performance — upward|
|PerformanceService subscribes to AchievementEarnedEvent|Performance watching Achievements — upward|
|Garment subscribes to StreakChangedEvent|Internal Achievements event — use AchievementService.getStreakRecord() instead|
|Garment calls ProgressionService|Garment watching Progression — upward|
|Analytics calls ProgressionService|Analytics watching Progression — upward|
|ProgressionService reads AchievementService directly|ProgressionService only knows about points — ScoringService handles achievements|
|Encouragement calls ProgressionService|Encouragement watching Progression — upward|
|AI Insights subscribes to AchievementEarnedEvent|AI Insights watching Achievements — upward|

---

## How to Use This Doc

**Before adding a subscription:** confirm the publishing feature is below the subscriber in the chain.

**Before adding a direct call:** confirm the called feature is below the caller in the chain.

**Before creating a new feature:** place it in the chain. Define its interface. Verify all dependencies point downward.

**When something feels wrong:** an upward dependency usually means the feature boundary is wrong. Redesign rather than accept the violation.

**Before modifying a locked feature:** treat it as a breaking change. All features above it depend on its interface — changes must be reviewed against every consumer.