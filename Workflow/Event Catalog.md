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

Published by: CommitmentIdentityService (Commitment feature) — detects day and week boundaries on `LongIntervalTick` using `TemporalHelper`.

```
WeekEndedEvent
  weekStart: DateTime     // start of the week that just ended

WeekStartedEvent
  weekStart: DateTime     // start of the new week
```

These are the signals that trigger weekly calculations across the app — cups, rewards, garment weekly sealing, AI summaries. Week boundaries are user-relative (week start day varies by region) — `TemporalHelper` makes the determination, `CommitmentIdentityService` publishes when the condition is met.

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
```

Structural change on instance — commitmentState, status, currentTarget, recurrence, activityWindow. Does not fire for `livePerformance` changes. Carries `CommitmentSnapshot` so subscribers have all data they need without a follow-up service call. `CommitmentSnapshot` defined in `service_commitment`.

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

One event covers all three write operations. `PerformanceService` recalculates identically regardless of type. `Encouragement` subscribes to `created` only. `value` is null on deleted.

No `instanceId` — `PerformanceService` identifies the instance from `definitionId` + `loggedAt` date.

All other downstream reactions (Rewards, Garment) watch `PerformanceUpdatedEvent` — not `ActivityEvent` directly.

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

## Reward Events

Published by: RewardService, StreakService

```
CupEarnedEvent
  cup: WeeklyCup

RewardEarnedEvent
  record: RewardRecord

StreakChangedEvent
  definitionId: String
  currentStreak: int
  bestStreak: int

GlobalBestStreakEvent
  newBest: int
  definitionId: String
  commitmentName: String
```

`GlobalBestStreakEvent` published by `StreakService` when a new personal best streak is set across all commitments.

---

## Achievement Events

Published by: AchievementService

```
AchievementEarnedEvent
  record: AchievementRecord
```

---

## Progression Events

Published by: ProgressionService

```
LevelReachedEvent
  newLevel: int
  levelName: String
  previousLevel: int

BonusThreadAwardedEvent
  weekStart: DateTime
  bonusAmount: double
  trigger: BonusTrigger
```

---

## Notes

**`livePerformance` changes** are signalled by `PerformanceUpdatedEvent`, not `InstanceUpdatedEvent`. The two event types have distinct meanings — structural instance changes vs performance value changes. Keeping them separate prevents features from receiving noise unrelated to their concern.

**Internal events** (`CommitmentEvent`) are not part of the application's public event surface. They exist only for coordination within the Commitment feature. No other feature should ever subscribe to them.

**Tick events carry timestamps only.** No feature ever publishes a `DayEndedEvent` or `MidnightEvent` from Heartbeat directly. Each feature that needs to react to a time boundary detects it independently using `TemporalHelper`.