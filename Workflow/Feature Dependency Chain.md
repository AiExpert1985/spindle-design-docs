**File Name**: feature_dependency_chain **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the authoritative reference for feature ordering and allowed dependencies. Check this before adding any new cross-feature call, event subscription, or public function.

---

## The Rule

Dependencies flow downward only. A feature may call services or subscribe to events from features below it. It may never depend on a feature above it. Any upward arrow is a design violation.

Lower features never know upper features exist. A lower feature publishes events and exposes service functions. It has no knowledge of who subscribes or who calls it. Upper features watch lower features — never the reverse. See `architecture_rules` §3 and §4.

**The test:** draw an arrow from the depending feature to the feature it depends on. All arrows must point downward.

---

## Domain Utilities

Not features. No lifecycle, no state, no events. Static utility classes and shared types available to any layer above Domain.

- **TickGuard** — idempotency utility for tick subscribers that run logic without consuming a record. See `tick_guard`.
- **Achievable** — interface implemented by domain models that participate in the achievement system. See `interface_achievable`.
- **AchievementRecord** — the unified display model for any earned achievement. See `model_achievement_record`.
- **AchievementType** — enum defining the category of an achievement (`cup`, `streakMilestone`, `reward`).

---

## Feature Chain

```
┌─────────────────────────────────────────────────────────┐
│  Heartbeat · Logger · NotificationService               │  base features — no domain knowledge
│  each independent, depended on by all above             │  depend on nothing
└─────────────────────────────────────────────────────────┘
      ↓
UserCore              (UserCoreProfile, preferences, tier — read by any feature)
      ↓
TemporalHelper        (answers time questions using user preferences)
      ↓
Notification Scheduling
  (watches Commitment instances, watches Heartbeat, fires via NotificationService)
      ↓
Commitment            ✓ LOCKED
      ↓
Activity              ✓ LOCKED
      ↓
Performance           ✓ LOCKED
      ↓
┌─────────────────────────────────────────────────────────┐
│  Streak · Cups · Rewards · Analytics                    │  same level — each independent
│  Streak publishes StreakChangedEvent                    │  no dependency between them
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│  Milestones                                             │  depends on Streak only
│  subscribes to StreakChangedEvent                       │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│  Garment · Achievements                                 │  same level — no dependency
│  Garment reads Streak (via AcceleratorService)          │  between them
│  Achievements subscribes to Cup, Reward, Milestone      │
└─────────────────────────────────────────────────────────┘
      ↓
Progression           (ScoringService · ProgressionService — internal)
      ↓
Encouragement
      ↓
┌─────────────────────────────────────────────────────────┐
│  AI Insights · Core                                     │  same level — no dependency
└─────────────────────────────────────────────────────────┘
      ↓
UserSettings          (writes UserCoreProfile, capability checks,
                       onboarding, subscription, referral)
```

---

## Key Design Principles

**Base features have no domain knowledge.** Heartbeat, Logger, and NotificationService provide raw capabilities. They never import, subscribe to, or call any feature above them. See `architecture_rules` §3.

**TemporalHelper centralizes time interpretation and boundary detection.** Any feature that needs to know whether a timestamp is a day boundary, week boundary, or within waking hours calls `TemporalHelperService`. Any feature that needs to react to a day or week boundary subscribes to `TemporalHelperService` events — never detects boundaries independently. It owns the `UserCoreService` dependency internally and subscribes to Heartbeat — no other feature needs to do either for time purposes.

**UserCore is the preference foundation.** Any feature that needs user preferences or tier reads from `UserCoreService`. Only `UserSettingsService` writes to it.

**Notification Scheduling sits above Commitment.** It subscribes to instance events from Commitment and calls NotificationService downward. It is fully removable — deleting it stops notifications, nothing else changes.

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

### Heartbeat

**Publishes:** `LongIntervalTick(timestamp)`, `ShortIntervalTick(timestamp)`

**Public functions (TickService):** `getLastTickTimestamp()`

No domain knowledge. No subscriptions. See `heartbeat`.

---

### Logger

**Public functions (LoggerService):** `debug(message, context?)`, `info(message, context?)`, `warning(message, context?)`, `error(AppError, stack?, context?)`

No events published. No subscriptions. See `logger`.

---

### NotificationService

**Public functions:** `push(message, route?)`

No events published. No subscriptions. See `notification_service`.

---

### UserCore

**Public functions (UserCoreService):** `getProfile()`, `getTier()`, `getTemporalPreferences()`, `getActivityWindowDefaults(recurrence)`, `watchProfile()`

No events published. No events subscribed. Pure read-only interface. See `service_user_core`, `repository_user_core`, `model_user_core_profile`.

---

### TemporalHelper

**Publishes:** `DayStartedEvent(dayStart)`, `DayEndedEvent(dayEnd)`, `WeekEndedEvent(weekStart)`, `WeekStartedEvent(weekStart)`

**Public functions (TemporalHelperService):** `isDayBoundary(timestamp)`, `isWeekBoundary(timestamp)`, `isRestDay(timestamp)`, `isWakingHours(timestamp)`, `currentDayStart(timestamp)`, `currentWeekStart(timestamp)`, `previousWeekStart(timestamp)`

**Subscribes to:** `Heartbeat.longIntervalTick`

**Calls directly:** `UserCoreService.getTemporalPreferences()`, `UserCoreService.watchProfile()`. See `service_temporal_helper`.

---

### Notification Scheduling

**Subscribes to:** instance change events (Commitment), `ShortIntervalTick` (Heartbeat)

**Calls directly:** `NotificationService.push()`, `TemporalHelperService` (for warning time formatting)

No events published. No public read functions — internal only. See `service_notification_scheduling`.

---

### Commitment ✓ LOCKED

**Publishes (public):** `InstanceCreatedEvent(instanceId, definitionId, name, windowStart)`, `InstanceUpdatedEvent(instanceId, definitionId, windowStart, snapshot)`, `InstancePermanentlyDeletedEvent(definitionId)`

**Publishes (internal only):** `CommitmentEvent` — consumed only by `CommitmentIdentityService`

**Public functions (CommitmentService):** `getDefinition()`, `watchActiveCommitments()`, `watchFrozenCommitments()`, `watchDeletedCommitments()`, `watchCompletedCommitments()`, `getStateTransitionLog()`, `getPortfolioSize()`, `getActiveCount()`, `getRecentlyCreated()`, `getCommitmentCountsByState()`

**Public functions (CommitmentIdentityService):** `getInstances()`, `watchInstancesForDay()`, `getCurrentInstance()`, `getInstanceForCommitmentOnDate()`, `getInstancesForDay()`, `getInstancesForWeek()`, `updateLivePerformance()`

---

### Activity ✓ LOCKED

**Publishes:** `ActivityEvent` (created | updated | deleted)

**Public functions:** `recordEntry()`, `editEntry()`, `deleteEntry()`, `getTotalLoggedForCommitmentOnDate()`, `getEntriesForDay()`, `getEntriesForWeek()`, `getEntriesForPeriod()`

**Subscribes to:** `InstancePermanentlyDeletedEvent` (Commitment)

---

### Performance ✓ LOCKED

**Publishes:** `PerformanceUpdatedEvent(instanceId, definitionId, windowStart, livePerformance)`

**Public functions:** `getPerformanceForPeriod()`, `getDayScore()`, `getCommitmentWeekScore()`, `getOverallWeekScore()`, `isWindowSuccess(livePerformance)`

**Subscribes to:** `InstanceCreatedEvent`, `InstanceUpdatedEvent` (Commitment), `ActivityEvent` (Activity)

**Calls directly:** `CommitmentIdentityService.updateLivePerformance()`, `CommitmentIdentityService.getInstanceForCommitmentOnDate()`, `CommitmentIdentityService.getInstances()`, `ActivityService.getTotalLoggedForCommitmentOnDate()`

---

### Streak

**Publishes:** `StreakChangedEvent(definitionId, currentStreak, bestStreak)`

**Public functions:** `getStreakRecord(definitionId)`, `watchStreakRecord(definitionId)`, `getBestStreakOverall()`

**Subscribes to:** `InstanceUpdatedEvent` where `status: closed` (Commitment), `InstancePermanentlyDeletedEvent` (Commitment), `WeekEndedEvent` (TemporalHelper)

**Calls directly:** `PerformanceService.isWindowSuccess()`, `PerformanceService.getCommitmentWeekScore()`, `CommitmentIdentityService.getInstancesForWeek()`

---

### Cups

**Implements Achievable on:** `WeeklyCup`

**Publishes:** `CupEarnedEvent(cup: WeeklyCup)`

**Public functions:** `getAllCups()`, `getCupsSince(from)`

**Subscribes to:** `WeekEndedEvent` (TemporalHelper)

**Calls directly:** `PerformanceService.getOverallWeekScore()`

---

### Rewards

**Implements Achievable on:** `RewardRecord`

**Publishes:** `RewardEarnedEvent(record: RewardRecord)`

**Public functions:** `getLatestReward()`

**Subscribes to:** `WeekEndedEvent` (TemporalHelper)

**Calls directly:** `PerformanceService.getPerformanceForPeriod()`

---

### Analytics

**Public functions:** `computeCommitmentFacts()`, `computeWeeklyFacts()`, `computeDayFacts()`

**Calls directly:** `ActivityService`, `PerformanceService`, `CommitmentIdentityService`, `StreakService.getStreakRecord()`

No events subscribed. No events published. Pure computation on demand.

---

### Milestones

**Implements Achievable on:** `MilestoneRecord`

**Publishes:** `MilestoneEarnedEvent(record: MilestoneRecord)`

**Subscribes to:** `StreakChangedEvent` (Streak), `InstancePermanentlyDeletedEvent` (Commitment)

**Calls directly:** `CommitmentService.getDefinition()` — name snapshot only

---

### Garment

**Publishes:** `GarmentUpdatedEvent(definitionId, completionPercent, weeklyDelta)`

**Public functions:** `watchGarmentProfile()`, `getWeeklyProgress()`, `getLiveWeekDelta()`

**Subscribes to:** `InstanceCreatedEvent` (Commitment), `PerformanceUpdatedEvent` (Performance), `InstancePermanentlyDeletedEvent` (Commitment), `WeekEndedEvent` (TemporalHelper)

**Calls directly:** `CommitmentService.getDefinition()`, `StreakService.getStreakRecord()` (via AcceleratorService)

---

### Achievements

**Publishes:** `AchievementEarnedEvent(record: AchievementRecord)`

**Public functions (AchievementService):** `getAchievements(from, to, type?)`, `watchRecentAchievements(from, to)`, `getCupHistory(from, to)`, `getCupsSince(from)`, `getStreakRecord(definitionId)`, `getBestStreakOverall()`

**Subscribes to:** `CupEarnedEvent` (Cups), `RewardEarnedEvent` (Rewards), `MilestoneEarnedEvent` (Milestones)

**Calls directly:** each event's model `.toAchievementRecord()` via Achievable interface; `CupService` proxied reads; `StreakService` proxied reads

---

### Progression

**Internal services:** `ScoringService` (`AchievementEarnedEvent` → `PointsAwardedEvent`), `ProgressionService` (`PointsAwardedEvent` → `LevelReachedEvent`)

**Publishes:** `LevelReachedEvent`, `PointsAwardedEvent` (internal)

**Public functions:** `getProgressionSummary()`, `watchProgressionSummary()`, `getCurrentWeekProjection()`, `isReferralUnlocked()`

**Subscribes to:** `AchievementEarnedEvent` (Achievements) — consumed by ScoringService

**Calls directly:** `PerformanceService.getOverallWeekScore()` — current week projection only

---

### Encouragement

**Subscribes to:** `ActivityEvent` (Activity), `InstanceUpdatedEvent` where `status: closed` (Commitment), `AchievementEarnedEvent` (Achievements), `LevelReachedEvent` (Progression), `LongIntervalTick` (Heartbeat)

**Calls directly:** `PerformanceService.getDayScore()`, `CommitmentIdentityService.getInstancesForDay()`, `AnalyticsService.computeDayFacts()`, `UserCoreService.getProfile()`, `UserSettingsService.recordEncouragementSent()`

---

### Analytics

**Public functions:** `computeCommitmentFacts()`, `computeWeeklyFacts()`, `computeDayFacts()`

No events published. No events subscribed. Pure computation on demand.

**Calls directly:** `ActivityService.getEntriesForPeriod()`, `ActivityService.getEntriesForDay()`, `PerformanceService.getPerformanceForPeriod()`, `PerformanceService.getDayScore()`, `CommitmentIdentityService.getInstances()`

---

### AI Insights

**Public functions (AIInsightService):** `generateMicroInsight(definitionId)`, `generateQuickSummary(weekStart)`, `generateDeepReport(from, to)`, `getLastInsight(type, definitionId?)`

**Subscribes to:** `WeekEndedEvent` (TemporalHelper) — for auto quick summary trigger

**Calls directly:** `AnalyticsService` (all three compute functions), `UserSettingsService.checkAndDecrementInsightQuota()`, `AIInsightRepository`

---

### Core

Presentation only. Screens and components that assemble data from multiple features. No services, no repositories, no models of their own.

---

### UserSettings

Top-of-chain feature. Depends on Commitment and Performance via `UserCapabilityService`.

**Public functions (UserSettingsService):** `updateProfile()`, `updateWeekStartDay()`, `updateRestDays()`, `updateDayBoundaryHour()`, `updateWakingHours()`, `updateCelebrationSettings()`, `updateWeeklyReportSettings()`, `updateWarningNotifications()`, `recordEncouragementSent()`, `checkAndDecrementInsightQuota()`, `completeOnboarding()`

**Public functions (UserCapabilityService):** `getBlockedReason()`, `canAddCommitment` (Riverpod provider)

**Calls directly:** `CommitmentService.getPortfolioSize()`, `CommitmentService.getActiveCount()`, `CommitmentService.getRecentlyCreated()`, `PerformanceService.getPerformanceForPeriod()`

---

## Violation Examples

|Violation|Why wrong|
|---|---|
|Any feature outside Commitment subscribes to `CommitmentEvent`|Internal event|
|Heartbeat, Logger, or NotificationService calls any feature|Base features are domain-agnostic — never call upward|
|Notification Scheduling calls `ActivityService` or `PerformanceService`|Above its position — scheduling only, no scoring|
|Cups calls `StreakService`|Same-level dependency|
|Rewards calls `CupService`|Same-level dependency|
|Streak subscribes to `CupEarnedEvent`|Same-level dependency|
|Milestones subscribes to `CupEarnedEvent`|Milestones depends on Streak only — Cup is same-level|
|Milestones calls `RewardService`|Same-level dependency|
|Achievements calls `ProgressionService`|Upward dependency|
|Garment calls `AchievementService`|Same-level dependency|
|`ProgressionService` reads `AchievementService` directly|ProgressionService only knows points — ScoringService handles achievements|
|Any feature implements `Achievable` on a service|Achievable belongs on domain models only|
|`WeeklyCup.toAchievementRecord()` calls a service|Must be pure — no side effects|
|Any feature reads temporal preferences from `UserSettingsService`|Read from `UserCoreService` only|
|Any feature calls `UserCoreService.getTemporalPreferences()` directly|Call `TemporalHelperService` — it owns that dependency|
|Any feature subscribes to `Heartbeat` for boundary detection|Subscribe to `TemporalHelperService` events instead — boundary detection is centralized there|
|Any feature publishes `WeekEndedEvent` or `DayEndedEvent`|These are published only by `TemporalHelperService`|
|`TemporalHelperService` imports any feature above it in the stack|Only calls `UserCoreService` and subscribes to `Heartbeat`|

---

## How to Use This Doc

**Before adding a subscription:** confirm the publishing feature is below the subscriber.

**Before adding a direct call:** confirm the called feature is below the caller.

**Before creating a new achievement-producing feature:** implement `Achievable` on its domain model. Publish an event carrying that model. `AchievementService` subscribes automatically — no registration needed.

**Before modifying a locked feature:** treat as a breaking change. Review all consumers.

**Before reading user preferences in any feature:** call `UserCoreService` — never `UserSettingsService`. The only exception is `UserSettingsService` itself.