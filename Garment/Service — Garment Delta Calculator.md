**File Name**: service_garment_delta_calculator **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** calculates how much the garment moves on a given day based on the day's performance value. Pure function — no side effects, no storage, no service calls. Commitment type has no effect on the calculation — the delta is always positive and proportional to performance. Visual interpretation of the delta is the renderer's responsibility.

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
) → double                     // raw delta — always positive. Caller applies lower bound of 0.0, no upper bound.
```

`commitmentType` is not a parameter — the delta formula is identical for Do and Avoid. All inputs passed in — no reads from storage, no service calls.

---

## Formula

```
delta = (performanceValue / 100) × dailyFullContribution
```

- Full kept day (100% performance) → `+dailyFullContribution` (3.0%)
- Partial day → proportional
- Missed day (0%) → 0.0 — nothing added, nothing removed

The same formula applies to both Do and Avoid commitments. `completionPercent` always starts at 0.0 and grows in one direction. The renderer receives `completionPercent` and `commitmentType` and decides what that number means visually — a growing garment for Do, a dissolving garment for Avoid.

---

## Configurable Constants

All constants in `AppConfig` — never hardcoded.

```
dailyFullContribution: 3.0
```

Set at 3.0% so that 21 consecutive days of full performance at the accelerator ceiling (1.5×) completes one garment cycle — roughly 4.5% × 21 days ≈ 94.5%. This makes near-perfect 21-day effort the intended completion time, reflecting the commonly cited habit-formation window. Will be tuned from real user data after launch.

---

## Rules

- Pure function — no storage reads, no service calls, no side effects
- `commitmentType` never passed in — the calculator has no knowledge of commitment type
- Returns raw positive delta — caller applies lower bound of 0.0, no upper bound
- Decay is deferred — see `feature_garment` Later Improvements