**File Name**: feature_dependency_chain **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the authoritative reference for feature ordering and allowed dependencies. Check this before adding any new cross-feature call, event subscription, or public function.

---

## The Rule

Dependencies flow downward only. A feature may call services or subscribe to events from features below it. It may never depend on a feature above it. Any upward arrow is a design violation.

Lower features never know upper features exist. A lower feature publishes events and exposes service functions. It has no knowledge of who subscribes or who calls it. Upper features watch lower features — never the reverse. See `architecture_rules` section 3.

**The test:** draw an arrow from the depending feature to the feature it depends on. All arrows must point downward.

---

## Feature Chain

```
Infrastructure        (EventBus, BackgroundScheduler+TickService,
                       TemporalHelper, TickGuard, ErrorService)
      ↓
UserCore              (UserCoreProfile, preferences, tier — read by any feature)
      ↓
Notification          (send, schedule, cancel — called by any feature above)
                       Publishes: NotificationActionEvent
      ↓
Commitment            ✓ LOCKED
  (includes: CommitmentNotificationSchedulerService)
      ↓
Activity              ✓ LOCKED
      ↓
Performance           ✓ LOCKED
      ↓
┌────────────────────────────────────────────────────────┐
│  Streak · Cups · Rewards · Analytics · Grace           │  same level — each independent
│  Streak publishes StreakChangedEvent                   │  no dependency between them
│  Grace subscribes to NotificationActionEvent,          │
│  InstancePermanentlyDeletedEvent                       │
└────────────────────────────────────────────────────────┘
      ↓
┌────────────────────────────────────────────────────────┐
│  Milestones                                            │  depends on Streak only
│  subscribes to StreakChangedEvent                      │
└────────────────────────────────────────────────────────┘
      ↓
┌────────────────────────────────────────────────────────┐
│  Garment · Achievements                                │  same level — no dependency
│  Garment reads Streak (via AcceleratorService)         │  between them
│  Achievements subscribes to Cup, Reward, Milestone     │
└────────────────────────────────────────────────────────┘
      ↓
Progression           (ScoringService · ProgressionService — internal)
      ↓
Encouragement
      ↓
┌────────────────────────────────────────────────────────┐
│  AI Insights · Core                                    │  same level — no dependency
└────────────────────────────────────────────────────────┘
      ↓
UserSettings          (writes UserCoreProfile, capability checks,
                       onboarding, subscription, referral)
```

---

## Key Design Principles

**Infrastructure is feature-agnostic.** Infrastructure never imports, subscribes to, or calls any feature. It provides mechanisms — ticks, event routing, delivery, logging — and nothing more. See `architecture_rules` section 11.

**UserCore is the preference foundation.** Any feature that needs user preferences or tier reads from `UserCoreService`. Only `UserSettingsService` writes to it.

**Notification is a delivery mechanism.** Features call `NotificationService.schedule()` and `NotificationService.send()` downward. The Notification feature publishes `NotificationActionEvent` when a user taps an action button — it never processes the action itself.

**Commitment's public interface is instance events.** `CommitmentEvent` is internal. No feature outside Commitment subscribes to it.

**Performance is the score authority.** No feature above Performance calculates scores independently.

**`Achievable` is the achievement contract.** Any feature whose domain model implements `Achievable` participates in the achievement system. `AchievementService` subscribes to their events and calls `toAchievementRecord()` — it never imports feature internals.

**Achievements is a pure aggregator.** It owns `AchievementRecord` storage and the unified read interface. It does not own the logic of what constitutes an achievement — that belongs to each producing feature.

**Progression never knows what an achievement is.** `ScoringService` translates `AchievementEarnedEvent` to points. `ProgressionService` only knows about points and levels.

**Same-level features are fully independent.** No feature calls or subscribes to a feature at the same level.

**UserSettings sits at the top.** It is the only writer of `UserCoreProfile`. Its `UserCapabilityService` depends on Commitment and Performance — placing it above everything else.

---

## Locked Features

**Commitment** — `CommitmentDefinition`, `CommitmentInstance`, `CommitmentService`, `CommitmentIdentityService`. Public interface: instance events and read functions below.

**Activity** — `LogEntry`, `ActivityService`, `ActivityRepository`. Public interface: `ActivityEvent` and read/write functions below.

**Performance** — `PerformanceService`. Public interface: `PerformanceUpdatedEvent` and query functions below.

---

## Feature Interfaces

---

### Infrastructure

**Publishes:** LongIntervalTickEvent, ShortIntervalTickEvent

**Shared contracts:** Achievable (interface), AchievementRecord (model), AchievementType (enum)

**Public functions:** EventBus.publish(), EventBus.on(), TemporalHelper, TickGuard, TickService.getLastTickTimestamp(), ErrorService.handle(), ErrorService.log()

---

### UserCore

**Public functions (UserCoreService):** getProfile(), getTier(), getTemporalPreferences(), getActivityWindowDefaults(recurrence), getGracePreferences(), watchProfile()

No events published. No events subscribed. Pure read-only interface.

---

### Notification

**Publishes:** NotificationActionEvent(actionId, params)

**Public functions (NotificationService):** send(payload), schedule(payload, fireAt), cancel(notificationId)

**Subscribes to:** ShortIntervalTickEvent (Infrastructure) — to fire due scheduled notifications

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

**Publishes:** StreakChangedEvent(definitionId, currentStreak, bestStreak), GlobalBestStreakEvent(newBest, definitionId, commitmentName)

**Public functions:** getStreakRecord(definitionId), watchStreakRecord(definitionId), getBestStreakOverall()

**Subscribes to:** PerformanceUpdatedEvent (Performance), InstancePermanentlyDeletedEvent (Commitment)

**Calls directly:** PerformanceService.isWindowSuccess()

---

### Cups

**Implements Achievable on:** `WeeklyCup`

**Publishes:** CupEarnedEvent(cup: WeeklyCup)

**Public functions:** getAllCups(), getCupsSince(from)

**Subscribes to:** WeekEndedEvent (Commitment — published by CommitmentIdentityService)

**Calls directly:** PerformanceService.getOverallWeekScore()

---

### Rewards

**Implements Achievable on:** `RewardRecord`

**Publishes:** RewardEarnedEvent(record: RewardRecord)

**Public functions:** getLatestReward()

**Subscribes to:** WeekEndedEvent (Commitment — published by CommitmentIdentityService)

**Calls directly:** PerformanceService.getPerformanceForPeriod()

---

### Analytics

**Public functions:** computeCommitmentFacts(), computeWeeklyFacts(), computeDayFacts()

**Calls directly:** ActivityService, PerformanceService, CommitmentIdentityService, StreakService.getStreakRecord()

No events subscribed. No events published. Pure computation on demand.

---

### Grace

**Public functions:** grantGrace(definitionId, windowDate)

**Subscribes to:** NotificationActionEvent where actionId == 'grant_grace' (Notification), InstancePermanentlyDeletedEvent (Commitment)

**Calls directly:** NotificationService.schedule(), UserCoreService.getGracePreferences()

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

**Subscribes to:** InstanceCreatedEvent (Commitment), PerformanceUpdatedEvent where isClosed: true (Performance), InstancePermanentlyDeletedEvent (Commitment), WeekEndedEvent (Commitment — published by CommitmentIdentityService)

---

### Achievements

**Publishes:** AchievementEarnedEvent(record: AchievementRecord)

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

**Subscribes to:** ActivityEvent (Activity), PerformanceUpdatedEvent where isClosed: true (Performance), AchievementEarnedEvent (Achievements), LevelReachedEvent (Progression), LongIntervalTickEvent (Infrastructure)

**Calls directly:** PerformanceService.getDayScore(), CommitmentIdentityService.getInstancesForDay(), AnalyticsService.computeDayFacts(), UserCoreService.getProfile() — celebration preferences, UserSettingsService.recordEncouragementSent()

---

### AI Insights

**Public functions:** generateMicroInsight(), generateQuickSummary(), generateDeepReport()

**Calls directly:** AnalyticsService (all reads)

---

### Core

Presentation only. Screens and components that assemble data from multiple features. No services, no repositories, no models of their own.

---

### UserSettings

Top-of-chain feature. Depends on Commitment and Performance via UserCapabilityService.

**Public functions (UserSettingsService):** updateProfile(), updateWeekStartDay(), updateRestDays(), updateDayBoundaryHour(), updateWakingHours(), updateCelebrationSettings(), updateWeeklyReportSettings(), updateWarningNotifications(), updateGraceSettings(), recordEncouragementSent(), checkAndDecrementInsightQuota(), completeOnboarding()

**Public functions (UserCapabilityService):** getBlockedReason(), canAddCommitment (Riverpod provider)

**Calls directly:** CommitmentService.getPortfolioSize(), CommitmentService.getActiveCount(), CommitmentService.getRecentlyCreated(), PerformanceService.getPerformanceForPeriod()

---

## Violation Examples

|Violation|Why wrong|
|---|---|
|Any feature outside Commitment subscribes to CommitmentEvent|Internal event|
|Infrastructure subscribes to any feature event|Infrastructure is feature-agnostic — never subscribes|
|TemporalHelper reads UserSettingsService|Should read UserCoreService — UserSettings is top-of-chain|
|Cups calls StreakService|Same-level dependency|
|Rewards calls CupService|Same-level dependency|
|Grace calls CommitmentIdentityService|Upward — Grace sits above Commitment|
|CommitmentNotificationSchedulerService calls GraceService|Upward — Grace is above Commitment|
|Milestones subscribes to CupEarnedEvent|Milestones depends on Streak only — Cup is same-level|
|Milestones calls RewardService|Same-level dependency|
|Achievements calls ProgressionService|Upward dependency|
|Garment calls AchievementService|Same-level dependency — Garment reads Streak directly|
|ProgressionService reads AchievementService directly|ProgressionService only knows points — ScoringService handles achievements|
|Any feature implements Achievable on a service|Achievable belongs on domain models only|
|WeeklyCup.toAchievementRecord() calls a service|Must be pure — no side effects|
|Any feature reads temporal preferences from UserSettingsService|Read from UserCoreService — or via TemporalHelper|

---

## How to Use This Doc

**Before adding a subscription:** confirm the publishing feature is below the subscriber.

**Before adding a direct call:** confirm the called feature is below the caller.

**Before creating a new achievement-producing feature:** implement `Achievable` on its domain model. Publish an event carrying that model. `AchievementService` subscribes automatically — no registration needed.

**Before modifying a locked feature:** treat as a breaking change. Review all consumers.

**Before reading user preferences in any feature:** call `UserCoreService` — never `UserSettingsService`. The only exception is `UserSettingsService` itself reading its own state.

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

**Publishes:** LongIntervalTickEvent, ShortIntervalTickEvent

**Shared contracts:** Achievable (interface), AchievementRecord (model), AchievementType (enum)

**Public functions:** EventBus.publish(), EventBus.on(), TemporalHelper, TickGuard, TickService.getLastTickTimestamp(), ErrorService.handle(), ErrorService.log()

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