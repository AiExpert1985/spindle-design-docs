**File Name**: event_catalog **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** quick reference for all events in the system — what each event carries and who publishes it. For who subscribes to each event, see `feature_dependency_chain`.

---

## Tick Events

Published by: Heartbeat

```
LongIntervalTick
  timestamp: DateTime

ShortIntervalTick
  timestamp: DateTime
```

Raw timestamps only. Subscribers use `TemporalHelper` to interpret. Heartbeat never publishes boundary events — boundaries are culturally relative and require user preferences to detect. Feature services detect boundaries using `TemporalHelper` and publish the appropriate domain events themselves.

---

## Boundary Events

Published by: TemporalHelperService — detects day and week boundaries by subscribing to `Heartbeat.longIntervalTick` and applying user preferences via its own calculation. Deduplication is handled internally using its own in-memory state.

```
DayStartedEvent
  dayStart: DateTime      // start of the new day per user's dayBoundaryHour

DayEndedEvent
  dayEnd: DateTime        // end of the day that just closed

WeekEndedEvent
  weekStart: DateTime     // start of the week that just ended

WeekStartedEvent
  weekStart: DateTime     // start of the new week
```

These are the signals that trigger time-based reactions across the app — cups, rewards, garment weekly sealing, AI summaries. All boundaries are user-relative (week start day, day boundary hour vary by region) — TemporalHelper owns this detection so no other feature needs to do it independently.

---

## Commitment Events — Internal Only

Published by: CommitmentService. Consumed only by CommitmentIdentityService. Never subscribed to outside the Commitment feature.

```
CommitmentEvent
  type: CommitmentEventType       // created | updated | deleted
  definitionId: String
  snapshot: CommitmentSnapshot?   // null when type == deleted
```

`CommitmentSnapshot` is defined in `service_commitment` — see that doc for fields.

---

## Instance Events

Published by: CommitmentIdentityService. Public interface of the Commitment feature.

```
InstanceCreatedEvent
  instanceId: String
  definitionId: String
  name: String
  windowStart: DateTime
```

New or recreated instance exists. Carries `name` so subscribers never need a definition lookup to display or construct messages.

```
InstanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  snapshot: CommitmentSnapshot
    name: String
    commitmentType: CommitmentType
    recurrence: Recurrence
    target: Target
    activityWindow: ActivityWindow
    commitmentState: CommitmentState
```

Structural change on instance — commitmentState, status, currentTarget, recurrence, activityWindow. Does not fire for `livePerformance` changes. Carries `CommitmentSnapshot` so subscribers have all data they need without a follow-up service call.

```
InstancePermanentlyDeletedEvent
  definitionId: String
```

All instances for this commitment permanently removed.

---

## Activity Events

Published by: ActivityService

```
ActivityEvent
  type: ActivityEventType   // created | updated | deleted
  definitionId: String
  loggedAt: DateTime
  value: double?            // null on deleted
```

One event covers all three write operations. `value` is null on deleted.

No `instanceId` — the correct instance is identified at read time from `definitionId` + `loggedAt` date.

---

## Performance Events

Published by: PerformanceService

```
PerformanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
  livePerformance: double
```

Published after every `livePerformance` change. Features that need to react to a closed window watch `InstanceUpdatedEvent` for `snapshot.status == closed` directly — the closed signal is a lifecycle event on the instance, not a performance calculation event.

---

## Achievement-Adjacent Events

The following events were removed as part of the achievement system redesign:

- `CupEarnedEvent` — replaced by `CupService._addAchievement()` direct call
- `RewardEarnedEvent` — Rewards feature deferred to Later Improvements
- `MilestoneEarnedEvent` — Milestones feature removed; milestone detection is internal to `StreakService`
- `GlobalBestStreakEvent` — replaced by `StreakService._addAchievement()` direct call
- `StreakChangedEvent` — existed only for `MilestoneService` to subscribe. With Milestones gone, no external feature needs to subscribe to streak changes. Removed entirely.

All achievement moments are now recorded via direct `AchievementService.addAchievement()` calls from producing services. The `AchievementEarnedEvent` (published by `AchievementService`) is the only achievement-related event in the system.

---

## Garment Events

Published by: GarmentService

```
GarmentUpdatedEvent
  definitionId: String
  completionPercent: double
```

`GarmentUpdatedEvent` consumed by the presentation layer for live display. When a garment completes, `GarmentService` calls `AchievementService.addAchievement()` directly — no event published for the achievement handoff.

---

Published by: AchievementService

```
AchievementEarnedEvent
  record: AchievementRecord
```

---

## Progression Events

Published by: ProgressionService (LevelReachedEvent), ScoringService (PointsAwardedEvent — internal only)

```
LevelReachedEvent
  newLevel: int
  levelName: String
  previousLevel: int

PointsAwardedEvent             // internal to Progression — never subscribed outside
  points: double
  subtype: String
  createdAt: DateTime
```

`LevelReachedEvent` consumed by `EncouragementService` for celebration and `NotificationSchedulingService` for the level-up notification. `PointsAwardedEvent` is internal — `ScoringService` publishes it, `ProgressionService` consumes it. No feature outside Progression subscribes to it.

---

## Notes

**`livePerformance` changes** are signalled by `PerformanceUpdatedEvent`, not `InstanceUpdatedEvent`. The two event types have distinct meanings — structural instance changes vs performance value changes. Keeping them separate prevents features from receiving noise unrelated to their concern.

**Internal events** (`CommitmentEvent`) are not part of the application's public event surface. They exist only for coordination within the Commitment feature. No other feature should ever subscribe to them.

**Tick events carry timestamps only.** Heartbeat fires raw timestamps — it never publishes boundary events. `TemporalHelperService` subscribes to Heartbeat, detects day and week boundaries using user preferences, and publishes the semantic boundary events (`DayStartedEvent`, `DayEndedEvent`, `WeekEndedEvent`, `WeekStartedEvent`). Any feature may subscribe to Heartbeat directly for interval-based work — but no feature detects day or week boundaries independently. That detection is centralized in TemporalHelper.