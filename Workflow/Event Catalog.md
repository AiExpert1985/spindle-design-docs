**File Name**: event_catalog **Feature**: Infrastructure **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** quick reference for all events in the system — what each event carries and who publishes it. For who subscribes to each event, see `feature_dependency_chain`. For how the event bus works, see `infrastructure_eventbus`.

---

## Tick Events

Published by: TickService

```
LongIntervalTickEvent
  timestamp: DateTime

ShortIntervalTickEvent
  timestamp: DateTime

DayEndedEvent
  date: Date

WeekEndedEvent
  weekStart: DateTime

WeekStartedEvent
  weekStart: DateTime
```

Raw timestamps only. Subscribers use TemporalHelper to interpret.

---

## Commitment Events — Internal Only

Published by: CommitmentService. Consumed only by CommitmentIdentityService. Never subscribed to outside the Commitment feature.

```
CommitmentEvent
  type: CommitmentEventType       // created | updated | deleted
  definitionId: String
  snapshot: CommitmentSnapshot?   // null when type == deleted
```

```
CommitmentSnapshot
  name: String
  commitmentType: CommitmentType
  recurrence: Recurrence
  target: Target
  activityWindow: ActivityWindow  // includes warningEnabled
  commitmentState: CommitmentState
```

---

## Instance Events

Published by: CommitmentIdentityService. Public interface of the Commitment feature.

```
InstanceCreatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
```

New or recreated instance exists.

```
InstanceUpdatedEvent
  instanceId: String
  definitionId: String
  windowStart: DateTime
```

Structural change on instance — commitmentState, status, currentTarget, recurrence, activityWindow. Does not fire for `livePerformance` changes.

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

One event covers all three write operations. PerformanceService recalculates identically regardless of type. Encouragement subscribes to `created` only. `value` is null on deleted.

No `instanceId` — PerformanceService identifies the instance from `definitionId` + `loggedAt` date. Grace-period logs work naturally with this design.

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
  isClosed: bool
```

`isClosed: true` signals the final result for a closed window. Features that act only on final results (Rewards, Encouragement) filter on this flag.

---

## Reward Events

Published by: RewardService

```
CupEarnedEvent
  cupLevel: CupLevel
  weekStart: DateTime
  pointValue: double

StreakMilestoneReachedEvent
  definitionId: String
  streakCount: int
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

**Internal events** (CommitmentEvent) are not part of the application's public event surface. They exist only for coordination within the Commitment feature. No other feature should ever subscribe to them.