**File Name**: service_reward **Feature**: Achievements (internal) **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** evaluates whether the user qualifies for an occasional reward based on sustained performance over a rolling period. Internal to Achievements.

---

## How Occasional Rewards Are Earned

Every `AppConfig.rewardIntervalDays` days (default: 30) since the last reward, `RewardService` evaluates average performance over that period. If the average meets `AppConfig.rewardPerformanceThreshold`, a reward is earned. This recognises sustained consistency that weekly cups may not fully capture — a user who earns modest cups every week for 30 days deserves recognition.

---

## Events Subscribed

### `WeekEndedEvent` → `_onWeekEnded(event)`

Checks whether a reward evaluation is due.

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
publish RewardEarnedInternalEvent(record)
```

---

## Events Published (internal)

```
RewardEarnedInternalEvent
  record: RewardRecord
```

Consumed only by `AchievementService`.

---

## Rules

- Internal to Achievements
- Evaluation triggered by `WeekEndedEvent` — runs at most once per week, fires reward only when interval and threshold are both met
- Idempotent by design — `getLatestReward()` prevents double-awarding within the same interval
- Both interval and performance threshold must be met — time alone is not enough

---

## Dependencies

- EventBus — subscribes to `WeekEndedEvent`; publishes `RewardEarnedInternalEvent`
- `RewardRepository` — reads and writes `RewardRecord`
- `PerformanceService.getPerformanceForPeriod()` — rolling period score
- `AppConfig` — `rewardIntervalDays`, `rewardPerformanceThreshold`