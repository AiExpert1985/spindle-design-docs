**File Name**: model_reward_record **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** a permanent record of one occasional reward earned. Written when the user meets the periodic performance condition. Never updated.

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

## Rules

- Append-only — never edited after creation
- Written only by `RewardService`
- One reward per qualifying period — `RewardService` checks idempotency before writing
- Not tied to a specific commitment — cross-commitment, like cups