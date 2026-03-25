**File Name**: repository_reward **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage for `RewardRecord`. Called only by `RewardService`.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveReward(record)`|RewardRecord|—|
|`getLatestReward()`|—|RewardRecord? — used to calculate next reward eligibility|
|`getAllRewards()`|—|List ordered by earnedAt desc|

---

## Firestore Path

```
/users/{userId}/rewards/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `RewardService`
- Append-only — never edited after creation
- `getLatestReward` used to determine when the next reward window opens