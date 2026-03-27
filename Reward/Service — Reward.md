**File Name**: service_reward **Feature**: Rewards **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** evaluates whether the user qualifies for an occasional reward based on sustained performance over a rolling period. An independent feature — not internal to Achievements. Publishes `RewardEarnedEvent` carrying a `RewardRecord` which implements `Achievable`.

---

## Events Subscribed

### `TemporalHelperService.onWeekEnded` → `_onWeekEnded(event)`

```
lastReward = RewardRepository.getLatestReward()
daysSinceLast = lastReward == null
  ? daysSinceFirstCommitment
  : daysBetween(lastReward.earnedAt, now)

if daysSinceLast < AppConfig.rewardIntervalDays: return

periodStart = now - AppConfig.rewardIntervalDays days
score = PerformanceService.getPerformanceForPeriod(periodStart, now)

if score < AppConfig.rewardPerformanceThreshold: return

record = RewardRecord(periodStart, periodEnd: now, averageScore: score)
RewardRepository.saveReward(record)
publish RewardEarnedEvent(record: record)
```

---

## Events Published

```
RewardEarnedEvent
  record: RewardRecord    // implements Achievable
```

Consumed by `AchievementService` which calls `record.toAchievementRecord()`.

---

## Read Functions

### `getLatestReward()` → RewardRecord?

Used internally for interval calculation.

---

## Rules

- Independent feature — not internal to Achievements
- Both interval and performance threshold must be met — time alone is not enough
- Idempotent by design — `getLatestReward()` prevents double-awarding
- `RewardRecord` implements `Achievable` — translation is the model's responsibility

---

## Dependencies

- `TemporalHelperService` — subscribes to `onWeekEnded`
- `RewardRepository` — reads and writes `RewardRecord`
- `PerformanceService.getPerformanceForPeriod()` — rolling period score
- `AppConfig` — `rewardIntervalDays`, `rewardPerformanceThreshold`