**File Name**: feature_dependency_chain **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 30-Mar-2026

---

**Purpose:** the authoritative reference for feature ordering and allowed dependencies. Check this before adding any new cross-feature call, event subscription, or public function.

---

## The Rule

Dependencies flow downward only. A feature may call services or subscribe to events from features below it. It may never depend on a feature above it. Any upward arrow is a design violation.

Lower features never know upper features exist. A lower feature publishes events and exposes service functions. It has no knowledge of who subscribes or who calls it. Upper features watch lower features — never the reverse. See `architecture_rules` §3 and §4.

**The test:** draw an arrow from the depending feature to the feature it depends on. All arrows must point downward.

---

## Domain Utilities

Not features. No lifecycle, no state, no events. Static utility classes available to any layer above Domain.

`AchievementRecord`, `AchievementType`, and `AchievementSubtype` are owned by the **Achievements feature** — not the Domain layer. Producing features reference them as downward dependencies since Achievements sits below them in the chain.

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
TemporalHelper        (answers time questions, publishes day/week boundary events)
      ↓
Commitment            ✓ LOCKED
      ↓
Activity              ✓ LOCKED
      ↓
CommitmentNotifications
  (watches Commitment instances, checks Activity before firing, fires via NotificationService)
      ↓
Performance           ✓ LOCKED
      ↓
Achievements          (addAchievement() entry point — producing features call downward)
      ↓
┌─────────────────────────────────────────────────────────┐
│  Streak · Cups · Analytics                              │  same level — each independent
│  Streak and Cups call AchievementService directly       │  no dependency between them
└─────────────────────────────────────────────────────────┘
      ↓
Garment               (depends on Performance, Streak via AcceleratorService)
                      (calls AchievementService on garment completion)
      ↓
Progression           (ScoringService · ProgressionService — internal)
                      (subscribes to AchievementEarnedEvent)
      ↓
┌─────────────────────────────────────────────────────────┐
│  AI Insights · Core                                     │  same level — no dependency
└─────────────────────────────────────────────────────────┘
      ↓
UserSettings          (writes UserCoreProfile, capability checks,
                       onboarding, subscription, referral)
      ↓
Encouragement         (subscribes to Activity, Commitment, Heartbeat;
                       calls Performance, UserCore, UserSettings;
                       top of chain — depends on everything below)
```

---

## Key Design Principles

**Base features have no domain knowledge.** Heartbeat, Logger, and NotificationService provide raw capabilities. They never import, subscribe to, or call any feature above them. See `architecture_rules` §3.

**TemporalHelper centralizes time interpretation and boundary detection.** Any feature that needs to know whether a timestamp is a day boundary, week boundary, or within waking hours calls `TemporalHelperService`. Any feature that needs to react to a day or week boundary subscribes to `TemporalHelperService` events — never detects boundaries independently. It owns the `UserCoreService` dependency internally and subscribes to Heartbeat — no other feature needs to do either for time purposes.

**UserCore is the preference foundation.** Any feature that needs user preferences or tier reads from `UserCoreService`. Only `UserSettingsService` writes to it.

**CommitmentNotifications sits above Activity.** It subscribes to instance events from Commitment, checks Activity before firing to avoid notifying a user who already logged, and calls NotificationService downward for delivery. It is fully removable — deleting it stops notifications, nothing else changes.

**Commitment's public interface is instance events.** `CommitmentEvent` is internal. No feature outside Commitment subscribes to it.

**Performance is the score authority.** No feature above Performance calculates scores independently.

**Achievements is the single entry point for all achievement recording.** Producing features (Cups, Streak, Garment) call `AchievementService.addAchievement()` directly via a private `_addAchievement()` function inside each service. Achievements sits below all producing features — they call downward into a stable API. Adding a new producing feature requires zero changes to `AchievementService`.

**Achievement detection is internal to each producing service.** Each producing service owns its own `_detectAchievements()` logic. `StreakService` detects milestone thresholds and global best internally — no separate Milestones feature needed. `GarmentService` detects completion transitions internally. Adding a new achievement condition means one new check inside the relevant service's detector — nothing else changes.

**AchievementService is the single source of truth for all achievement data.** Every achievement flows through `addAchievement()`. All queries — cups history, streak records, best streak, garment completions — are answered from the `AchievementRecord` collection. No producing feature needs to be called for achievement data.

**Progression never knows what an achievement is.** `ScoringService` translates `AchievementEarnedEvent` to points using a subtype-to-points table in `AppConfig`. `ProgressionService` only knows about points and levels. Changing point values or adding new achievement types never touches `ProgressionService`.

**Same-level features are fully independent.** No feature calls or subscribes to a feature at the same level.

**UserSettings is the only writer of `UserCoreProfile`.** Its `UserCapabilityService` depends on Commitment and Performance — placing it near the top.

**Encouragement sits at the top of the chain.** It subscribes to Activity, Commitment, and Heartbeat events, reads from Performance and UserCore, and calls `UserSettingsService` for encouragement tracking. It depends on more features than any other, which is why it sits highest. It is fully removable — deleting it stops all encouragement signals, nothing else changes.

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

**Public functions (UserCoreService):** `getProfile()`, `getTier()`, `getTemporalPreferences()`, `watchProfile()`

No events published. No events subscribed. Pure read-only interface. See `service_user_core`, `repository_user_core`, `model_user_core_profile`.

---

### TemporalHelper

**Publishes:** `DayStartedEvent(dayStart)`, `DayEndedEvent(dayEnd)`, `WeekEndedEvent(weekStart)`, `WeekStartedEvent(weekStart)`

**Public functions (TemporalHelperService):** `isDayBoundary(timestamp)`, `isWeekBoundary(timestamp)`, `isRestDay(timestamp)`, `isWakingHours(timestamp)`, `currentDayStart(timestamp)`, `currentWeekStart(timestamp)`, `previousWeekStart(timestamp)`

**Subscribes to:** `Heartbeat.longIntervalTick`

**Calls directly:** `UserCoreService.getTemporalPreferences()`, `UserCoreService.watchProfile()`. See `service_temporal_helper`.

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

**Subscribes to:** `InstancePermanentlyDeletedEvent` (CommitmentIdentityService)

---

### CommitmentNotifications

**Subscribes to:** `InstanceCreatedEvent`, `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent` (Commitment), `ShortIntervalTick` (Heartbeat)

**Calls directly:** `ActivityService.getTotalLoggedForCommitmentOnDate()`, `NotificationService.push()`, `TemporalHelperService` (for warning lead time formatting)

No events published. No public read functions — internal only. See `service_commitment_notifications`.

---

### Performance ✓ LOCKED

**Publishes:** `PerformanceUpdatedEvent(instanceId, definitionId, windowStart, livePerformance)`

**Public functions:** `getPerformanceForPeriod()`, `getDayScore()`, `getCommitmentWeekScore()`, `getOverallWeekScore()`, `isWindowSuccess(livePerformance)`

**Subscribes to:** `InstanceCreatedEvent` (Commitment), `ActivityEvent` (Activity)

**Calls directly:** `CommitmentIdentityService.updateLivePerformance()`, `CommitmentIdentityService.getInstanceForCommitmentOnDate()`, `CommitmentIdentityService.getInstances()`, `ActivityService.getTotalLoggedForCommitmentOnDate()`

---

### Achievements

**Publishes:** `AchievementEarnedEvent(record: AchievementRecord)`

**Public functions (AchievementService):** `addAchievement(record)`, `getAchievements(from, to, type?, subtype?, definitionId?)`, `getAchievementsForWeek(weekStart)`, `getAchievementsForMonth(month)`, `getAchievementsForPeriod(from, to)`, `watchAchievements(from, to)`

No event subscriptions — producing features call `addAchievement()` directly downward.

---

### Streak

**Public functions:** `getStreakRecord(definitionId)`, `watchStreakRecord(definitionId)`, `getBestStreakOverall()`

**Subscribes to:** `InstanceUpdatedEvent` where `status: closed` (CommitmentIdentityService), `InstancePermanentlyDeletedEvent` (CommitmentIdentityService), `WeekEndedEvent` (TemporalHelper)

**Calls directly:** `PerformanceService.isWindowSuccess()`, `PerformanceService.getCommitmentWeekScore()`, `CommitmentIdentityService.getInstancesForWeek()`, `AchievementService.addAchievement()`

No public events published. Streak changes are internal. Achievement detection (milestone thresholds, global best) happens inside `_detectAchievements()` — internal to this service.

---

### Cups

**Public functions:** `getAllCups(from, to, definitionId?)`

**Subscribes to:** `WeekEndedEvent` (TemporalHelper)

**Calls directly:** `PerformanceService.getOverallWeekScore()`, `AchievementService.addAchievement()`

---

### Garment

**Publishes:** `GarmentUpdatedEvent(definitionId, completionPercent)`

**Public functions:** `watchGarmentProfile()`

**Subscribes to:** `InstanceCreatedEvent`, `InstancePermanentlyDeletedEvent` (CommitmentIdentityService), `PerformanceUpdatedEvent` (Performance)

**Calls directly:** `CommitmentService.getDefinition()`, `StreakService.getStreakRecord()` (via AcceleratorService), `AchievementService.addAchievement()`

---

### Progression

**Internal services:** `ScoringService` (subscribes to `AchievementEarnedEvent`, calls `ProgressionService.awardPoints()` directly), `ProgressionService` (handles level state, publishes `LevelReachedEvent`)

**Publishes:** `LevelReachedEvent(newLevel, levelName, previousLevel)`

**Public functions (ProgressionService):** `getProgressionSummary()`, `watchProgressionSummary()`, `getCurrentWeekProjection()`, `isReferralUnlocked()`

**Subscribes to:** `AchievementEarnedEvent` (Achievements) — consumed by ScoringService internally

**Calls directly:** `PerformanceService.getOverallWeekScore()` — current week projection only

---

### Encouragement

**Subscribes to:** `longIntervalTick` (Heartbeat), `ActivityEvent` (ActivityService), `InstanceUpdatedEvent` where `status: closed` (CommitmentIdentityService)

**Calls directly:** `PerformanceService.getDayScore()`, `PerformanceService.getPerformanceForPeriod()`, `CommitmentIdentityService.getInstancesForDay()`, `UserCoreService.getProfile()`, `UserSettingsService.recordEncouragementSent()`, `UserSettingsService.getLastEncouragementType()`

---

### Analytics

**Public functions:** `computeCommitmentFacts()`, `computeWeeklyFacts()`, `computeDayFacts()`

No events published. No events subscribed. Pure computation on demand.

**Calls directly:** `ActivityService.getEntriesForPeriod()`, `ActivityService.getEntriesForDay()`, `PerformanceService.getPerformanceForPeriod()`, `PerformanceService.getDayScore()`, `CommitmentIdentityService.getInstances()`

---

### AI Insights

**Public functions (AIInsightService):** `generateMicroInsight(definitionId)`, `generateQuickSummary(weekStart)`, `generateDeepReport(from, to)`, `getLastInsight(type, definitionId?)`

No event subscriptions — insights are generated on demand only.

**Calls directly:** `AnalyticsService` (all three compute functions), `UserSettingsService.checkAndDecrementInsightQuota()`, `AIInsightRepository`

---

### Core

Presentation only. Screens and components that assemble data from multiple features. No services, no repositories, no models of their own.

---

### UserSettings

Top-of-chain feature. Depends on Commitment and Performance via `UserCapabilityService`.

**Public functions (UserSettingsService):** `updateProfile()`, `updateWeekStartDay()`, `updateRestDays()`, `updateDayBoundaryHour()`, `updateWakingHours()`, `updateCelebrationSettings()`, `updateWeeklyReportSettings()`, `updateWarningNotifications()`, `recordEncouragementSent()`, `getLastEncouragementType()`, `checkAndDecrementInsightQuota()`, `completeOnboarding()`

**Public functions (UserCapabilityService):** `getBlockedReason()`, `canAddCommitment` (Riverpod provider)

**Subscribes to (UserCapabilityService):** `InstanceCreatedEvent`, `InstanceUpdatedEvent`, `InstancePermanentlyDeletedEvent` (CommitmentIdentityService), `DayStartedEvent` (TemporalHelper)

**Calls directly:** `CommitmentIdentityService.getPortfolioSize()`, `CommitmentIdentityService.getActiveCount()`, `CommitmentIdentityService.getRecentlyCreated()`, `PerformanceService.getPerformanceForPeriod()`

---

## Violation Examples

|Violation|Why wrong|
|---|---|
|Any feature outside Commitment subscribes to `CommitmentEvent`|Internal event|
|Heartbeat, Logger, or NotificationService calls any feature|Base features are domain-agnostic — never call upward|
|`CommitmentNotifications` calls `PerformanceService`|Above its position — notification logic only, no scoring|
|Cups calls `StreakService`|Same-level dependency|
|Any producing feature skips `AchievementService` and writes `AchievementRecord` directly|All achievements flow through `addAchievement()` — no exceptions|
|`AchievementService` calls any producing feature for reads|All reads are self-contained from its own collection|
|A separate service subscribes to streak changes to detect milestones|No `StreakChangedEvent` exists — milestone detection is internal to `StreakService._detectAchievements()`|
|Garment calls `ProgressionService`|Upward dependency|
|`ProgressionService` reads `AchievementService` directly|ProgressionService only knows points — ScoringService handles the translation|
|Any feature reads temporal preferences from `UserSettingsService`|Read from `UserCoreService` only|
|Any feature calls `UserCoreService.getTemporalPreferences()` directly|Call `TemporalHelperService` — it owns that dependency|
|Any feature subscribes to `Heartbeat` for boundary detection|Subscribe to `TemporalHelperService` events instead|
|Any feature publishes `WeekEndedEvent` or `DayEndedEvent`|Published only by `TemporalHelperService`|
|`TemporalHelperService` imports any feature above it in the stack|Only calls `UserCoreService` and subscribes to `Heartbeat`|

---

## How to Use This Doc

**Before adding a subscription:** confirm the publishing feature is below the subscriber.

**Before adding a direct call:** confirm the called feature is below the caller.

**Before creating a new achievement-producing feature:** construct an `AchievementRecord` in the service at the achievement moment. Call `AchievementService.addAchievement()` downward. Add one `AchievementSubtype` enum value and one `AppConfig` point entry. No changes to `AchievementService` needed.

**Before modifying a locked feature:** treat as a breaking change. Review all consumers.

**Before reading user preferences in any feature:** call `UserCoreService` — never `UserSettingsService`. The only exception is `UserSettingsService` itself.