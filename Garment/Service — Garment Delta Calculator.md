**File Name**: service_garment_delta_calculator **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** calculates how much the garment moves on a given day based on the day's performance value. Pure function — no side effects, no storage, no service calls.

Abstract interface — the formula inside can be replaced entirely without touching `GarmentService` or any other caller.

---

## Abstraction

```
GarmentDeltaCalculator (abstract)
  → DefaultGarmentDeltaCalculator    // proportional contribution implementation
  → (future: science-backed, AI-tuned, or user-configured)
```

---

## Interface

```
calculate(
  performanceValue: double,    // already accelerator-modified by caller
  commitmentType: CommitmentType,
) → double                     // raw delta — caller clamps to 0.0–100.0
```

All inputs passed in — no reads from storage, no service calls. The caller (`GarmentService`) applies the acceleration before passing `performanceValue` in.

---

## Default Implementation — Formula

### Do Commitment

```
delta = (performanceValue / 100) × dailyFullContribution
```

- Full kept day (100% performance) → `+dailyFullContribution` (default: 1.5%)
- Partial day → proportional
- Missed day (0%) → 0.0 — nothing added, nothing removed

### Avoid Commitment

```
Successful avoided day  →  -(performanceValue / 100) × dailyFullContribution
Breach day              →  +(breachPenalty)    // moves back toward 100%
```

Direction is reversed — a successful avoid day moves `completionPercent` toward 0.0 (unraveled).

---

### Avoid Direction Table

|Event|Do habit|Avoid habit|
|---|---|---|
|Successful day|percent increases|percent decreases|
|Missed / breach|no change (0 delta)|percent increases|
|Starting value|0.0%|100.0%|
|Goal|reach 100.0%|reach 0.0%|

---

## Configurable Constants

All constants in `AppConfig` — never hardcoded.

```
dailyFullContribution: 1.5
dailyBreachPenalty:    2.0
```

Starting values. Will be tuned from real user data after launch.

---

## Rules

- Pure function — no storage reads, no service calls, no side effects
- All inputs passed in by `GarmentService` — never fetched internally
- Returns raw delta — `GarmentService` clamps the result to 0.0–100.0
- Decay is deferred — see `feature_garment` Later Improvements