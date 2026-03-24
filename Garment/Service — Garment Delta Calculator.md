**File Name**: service_garment_delta_calculator **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** calculates how much the garment moves on a given day based on the commitment's performance result. Pure function — given current state and today's result, returns the delta. No side effects, no storage.

Abstract interface — the formula inside can be replaced entirely without touching `GarmentService` or any other caller.

---

## Abstraction

```
GarmentDeltaCalculator (abstract)
  → DefaultGarmentDeltaCalculator    // rule-based implementation
  → (future: science-backed, AI-tuned, or user-configured)
```

---

## Interface

```
calculate(
  completionPercent: double,
  consecutiveFailDays: int,
  livePerformance: double,
  commitmentType: CommitmentType,
  currentStreak: int,
) → GarmentDelta
```

```
GarmentDelta
  delta: double                // net change to apply to completionPercent
  updatedConsecutiveFailDays: int
```

All inputs are passed in — no reads from storage, no service calls.

---

## Default Implementation — Formula

### Daily Contribution

The base movement for a kept day.

**Do commitment:**
- Full kept day (100%) → `+dailyFullContribution` (default: 1.5%)
- Partial day → proportional: `livePerformance / 100 × dailyFullContribution`
- Missed day (0%) → 0 contribution, does not trigger decay alone

**Avoid commitment:**
- Successful avoided day → `-dailyFullContribution` (moves toward 0%)
- Breach day → `+dailyBreachPenalty` (default: 2.0%, moves back toward 100%)

---

### Decay

Applies when consecutive failures exceed the grace threshold.

- Condition: `consecutiveFailDays >= decayThreshold` (default: 2)
- Amount: `-decayAmount` per day (default: 1.0%)
- Direction reversed for Avoid commitments
- Any successful day resets `consecutiveFailDays` to 0

The grace threshold means one or two bad days have no decay penalty — a pattern of failure is required. This reflects real behavioral science: occasional misses are normal, sustained failure is the signal.

---

### Streak Bonus

Applies when a streak is active.

- Condition: `currentStreak >= streakBonusThreshold` (default: 3 days)
- Effect: daily contribution multiplied by `streakBonusMultiplier` (default: 1.2)
- Direction: same as daily contribution — Do habits weave faster, Avoid habits unweave faster

---

### Avoid Direction

All calculations work identically for both commitment types — the direction is reversed for Avoid.

| Event | Do habit | Avoid habit |
|---|---|---|
| Successful day | percent increases | percent decreases |
| Breach / miss | percent unchanged (decay may apply) | percent increases |
| Decay active | percent decreases | percent increases |
| Streak bonus | increases faster | decreases faster |
| Starting value | 0.0% | 100.0% |
| Goal | reach 100.0% | reach 0.0% |

---

## Configurable Constants

All constants are in `AppConfig` — never hardcoded.

```
dailyFullContribution: 1.5
dailyBreachPenalty: 2.0
decayThreshold: 2
decayAmount: 1.0
streakBonusThreshold: 3
streakBonusMultiplier: 1.2
```

These are starting values. They will be tuned based on real user data after launch. The abstraction exists precisely so tuning requires changing one file, not touching any formula.

---

## Rules

- Pure function — no storage reads, no service calls, no side effects
- All inputs passed in by `GarmentService` — never fetched internally
- Returns `GarmentDelta` — `GarmentService` applies it and clamps the result to 0.0–100.0
- Clamping is the service's responsibility — the calculator returns the raw delta
