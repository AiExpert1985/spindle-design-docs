**File Name**: service_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the single public interface of the Achievements feature. Subscribes to all internal achievement events, translates them into unified `AchievementRecord` entries, and publishes one public event consumed by the rest of the application. External features read achievement data only through this service.

---

## Design Decisions

**Why one public facade over multiple exposed services.** Internal services (`CupService`, `MilestoneService`, `RewardService`, `StreakService`) are implementation details. If the cup calculation changes, or milestones are restructured, nothing outside Achievements needs to change. The facade is the contract — stable regardless of internal changes.

**Why one public event instead of separate typed events.** `AchievementEarnedEvent` carries an `AchievementRecord` which contains `type` and `subtype`. Consumers read what they need from the record. Adding a new achievement type requires no new event — just a new internal service that writes to `AchievementRecord` and triggers `AchievementEarnedEvent`. `ScoringService` (Progression) reads `type` and `subtype` to look up point values. Encouragement reads `type` to decide on celebration style.

**Why AchievementRecord is a separate model rather than using source models directly.** Each internal service has its own model (`WeeklyCup`, `MilestoneRecord`, `RewardRecord`). A unified display model means the achievements screen and external consumers work with one consistent format. The source models remain internal — external features never need to know about them.

---

## Events Subscribed (internal)

### `CupEarnedInternalEvent` → `_onCupEarned(event)`

```
record = AchievementRecord(
  type: cup,
  subtype: event.cup.cupLevel.name,
  sourceId: event.cup.id,
  definitionId: null,
  earnedAt: event.cup.createdAt
)
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `MilestoneReachedInternalEvent` → `_onMilestoneReached(event)`

```
record = AchievementRecord(
  type: streakMilestone,
  subtype: '${event.record.streakCount}_day',
  sourceId: event.record.id,
  definitionId: event.record.definitionId,
  earnedAt: event.record.earnedAt
)
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `RewardEarnedInternalEvent` → `_onRewardEarned(event)`

```
record = AchievementRecord(
  type: reward,
  subtype: 'periodic',
  sourceId: event.record.id,
  definitionId: null,
  earnedAt: event.record.earnedAt
)
AchievementRepository.saveRecord(record)
publish AchievementEarnedEvent(record)
```

### `InstancePermanentlyDeletedEvent` → `_onDeleted(event)`

Cleans up `AchievementRecord` entries linked to this `definitionId`.

---

## Events Published (public)

```
AchievementEarnedEvent
  record: AchievementRecord
```

The single public event of the Achievements feature. Consumed by `ScoringService` (Progression) for point assignment and `Encouragement` for celebration triggers.

---

## Public Read Functions

### `getAchievements(limit?, type?)` → List<AchievementRecord>

Ordered by earnedAt descending. Optional type filter. Used by the achievements screen.

### `watchRecentAchievements(limit?)` → Stream<List<AchievementRecord>>

Live stream. Used by the achievements screen for real-time updates.

### `getCupHistory()` → List<WeeklyCup>

Full cup history ordered by weekStart descending. Proxied from `CupService`. Used by the Progression screen cup breakdown.

### `getCupsSince(from)` → List<WeeklyCup>

Proxied from `CupService`. Used by `ScoringService` for bonus trigger evaluation.

### `getStreakRecord(definitionId)` → StreakRecord?

Proxied from `StreakService`. Used by `AcceleratorService` (Garment) and the commitment detail screen.

### `getBestStreakOverall()` → int

Proxied from `StreakService`. Used by the Your Record screen.

---

## Rules

- The only public interface of the Achievements feature — no feature outside calls internal services directly
- Writes `AchievementRecord` only — never modifies internal service records
- `AchievementEarnedEvent` published only after the record is successfully written
- Idempotency delegated to internal services — each internal service prevents duplicate records before publishing its internal event

---

## Dependencies

- EventBus — subscribes to internal events; publishes `AchievementEarnedEvent`
- `AchievementRepository` — writes and reads `AchievementRecord`
- `CupService` — proxied reads
- `StreakService` — proxied reads