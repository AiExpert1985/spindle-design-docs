**File Name**: service_achievement **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** the single entry point for recording achievements. Producing features call `addAchievement()` directly. Stores `AchievementRecord`, publishes `AchievementEarnedEvent`, and serves all achievement reads from its own collection.

---

## Public Write Function

### `addAchievement(AchievementRecord record)`

```
AchievementRepository.saveRecord(record)   // upsert by id — idempotent by design
publish AchievementEarnedEvent(record)
```

Idempotency is structural — the producing service generates a deterministic `id` for each achievement moment. Writing the same ID twice produces the same document. No existence check needed.

---

## Events Published

```
AchievementEarnedEvent
  record: AchievementRecord
```

Published after every successful write. Consumed by features above that score points or respond to achievement moments.

---

## Read Functions

### `getAchievements(from, to, type?, subtype?, definitionId?)` → List<AchievementRecord>

Unified read. All filters optional. Ordered by `createdAt` descending.

### Convenience wrappers

```dart
getAchievementsForWeek(DateTime weekStart)
getAchievementsForMonth(DateTime month)
getAchievementsForPeriod(DateTime from, DateTime to)
```

### `watchAchievements(from, to)` → Stream<List<AchievementRecord>>

Live stream for the achievements screen during a session.

---

## Rules

- `addAchievement()` is the only write path — no other service writes `AchievementRecord`
- `AchievementEarnedEvent` published only after successful write
- No event subscriptions — all producing features call directly
- All reads are self-contained — no calls to other features
- Records are permanent — never deleted for any reason
- All reads use a time window
- Idempotency guaranteed by deterministic `id` — no existence check needed

---

## Dependencies

- `AchievementRepository` — reads and writes `AchievementRecord`
- Publishes `AchievementEarnedEvent` on its own stream