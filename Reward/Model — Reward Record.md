**File Name**: model_reward_record **Feature**: Rewards **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** a permanent record of one occasional reward earned. Implements `Achievable` so `AchievementService` can build a unified record from it when `RewardEarnedEvent` arrives.

---

## What an Occasional Reward Is

Every `AppConfig.rewardIntervalDays` days (default: 30) of sustained performance above threshold, the user earns an occasional reward. Unlike cups which evaluate one week at a time, the reward evaluates a rolling window of consistent effort over a longer period. It recognises sustained commitment that weekly cups may not fully capture.

---

## Fields

```
RewardRecord
  id: String
  periodStart: DateTime    // start of the evaluation period
  periodEnd: DateTime      // end of the evaluation period
  averageScore: double     // average performance over the period
  earnedAt: DateTime
  createdAt: DateTime
```

- **periodStart / periodEnd** — the rolling window that qualified for this reward.
- **averageScore** — stored for auditability. The score that earned the reward.

---

## Achievable Implementation

```dart
@override
AchievementRecord toAchievementRecord() {
  return AchievementRecord(
    type: AchievementType.reward,
    subtype: 'periodic',
    sourceId: id,
    definitionId: null,     // rewards are cross-commitment
    earnedAt: earnedAt,
  );
}
```

Pure function — no side effects, no service calls.

---

## Rules

- Append-only — never edited after creation
- Written only by `RewardService`
- One reward per qualifying period — `RewardService` checks idempotency
- `toAchievementRecord()` called only by `AchievementService`