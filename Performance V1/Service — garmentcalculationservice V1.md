
**Created**: 18-Mar-2026 
**Modified**: - 
**Feature**: Commitment 
**Phase**: 2

**Purpose:** owns all garment completion percentage logic — daily performance contribution, decay, streak bonus, and weekly delta calculation. Abstract interface so the internal rules can be replaced or tuned without touching anything that calls this service or the garment display component.

---

## Abstraction

```
GarmentCalculationService (abstract)
  → DefaultGarmentCalculationService    (Phase 2 implementation — rule-based)
  → (future: science-backed rules when evidence is available)
```

The interface is fixed. The rules inside the implementation can be rewritten completely — better science, tuned numbers, AI-informed thresholds — without changing any caller or any display component.

---

## Separation of Concerns

This service owns **one number**: `garmentCompletionPercent` on `CommitmentDefinition`.

It does not own `progressPercent` on `CommitmentInstance` — that belongs to `ScoringService`.

These are two separate values with two separate purposes:

- `progressPercent` — how much of today's instance target was achieved. Used by scoring, cups, dashboard.
- `garmentCompletionPercent` — how complete the garment is as a long-term accumulation. Used only by the garment display.

They are related but not the same. A single excellent day moves `progressPercent` significantly. It moves `garmentCompletionPercent` by a small amount — one thread added to a long-term weave.

---

## Core Function

### `calculateAndSaveDailyUpdate(definitionId, date)`

The single entry point for all garment updates. Called once per day per active commitment by the midnight background task.

**Steps in order:**

```
1. Read current CommitmentDefinition (garmentCompletionPercent, commitmentType,
   currentStreak, consecutiveFailDaysForDecay, isCompleted, isFrozen)

2. If isFrozen or isCompleted → exit immediately, no update

3. Read today's CommitmentInstance for this definition
   → if no instance exists for today → treat as missed

4. Apply daily performance
   → applyDailyPerformance(currentPercent, instanceResult, commitmentType)
   → returns updatedPercent

5. Apply decay if eligible
   → applyDecay(updatedPercent, consecutiveFailDaysForDecay, commitmentType)
   → returns updatedPercent, updatedConsecutiveFailDays

6. Apply streak bonus if eligible
   → applyStreakBonus(updatedPercent, currentStreak, commitmentType)
   → returns finalPercent

7. Clamp result: do habits → 0.0 to 100.0, avoid habits → 0.0 to 100.0
   (avoid habits start at 100.0 and move toward 0.0 — clamping is the same)

8. Save updated garmentCompletionPercent and consecutiveFailDaysForDecay
   to CommitmentDefinition via CommitmentService.saveDefinitionState()

9. Update live week CommitmentWeeklyProgress record
   → updateLiveWeekDelta(definitionId, date)
```

Idempotent — checks `lastGarmentUpdateDate` before running. Safe to call multiple times for the same date.

Called by: midnight background scheduler action.

---

## Sub-Functions

### `applyDailyPerformance(currentPercent, instanceResult, commitmentType)`

Pure function. Calculates how much the garment moves based on today's instance result.

**Do commitment:**

- `progressPercent` of today's instance contributes a proportional gain
- Full kept day (100%) → adds `dailyFullContribution` (configurable constant, default: 1.5%)
- Partial day → adds proportionally: `progressPercent / 100 × dailyFullContribution`
- Missed day (0%) → adds 0, but does not trigger decay on its own

**Avoid commitment:**

- Successful avoided day (instance kept) → subtracts `dailyFullContribution` (moves toward 0%)
- Breach day → adds `dailyBreachPenalty` (configurable constant, default: 2.0%)

Returns: `double` — the updated percent after performance is applied.

---

### `applyDecay(currentPercent, consecutiveFailDays, commitmentType)`

Pure function. Applies decay if the 2-day consecutive failure window is active.

**Decay condition:** `consecutiveFailDays >= 2`

**Decay amount:** `decayAmount` (configurable constant, default: 1.0%) removed per day once condition is met.

**Direction:**

- Do commitment: decay reduces percent (garment loses threads)
- Avoid commitment: decay increases percent (bad habit reasserts)

**consecutiveFailDays update:**

- Today was a success → reset to 0
- Today was a failure → increment by 1

Returns: `{ updatedPercent: double, updatedConsecutiveFailDays: int }`

---

### `applyStreakBonus(currentPercent, currentStreak, commitmentType)`

Pure function. Applies a small multiplier to the daily performance contribution when a streak is active.

**Bonus condition:** `currentStreak >= streakBonusThreshold` (configurable, default: 3 days)

**Effect:** the performance contribution calculated in `applyDailyPerformance` is multiplied by `streakBonusMultiplier` (configurable, default: 1.2 — 20% faster weaving on streak days).

**Direction:** same as daily performance — do habits weave faster, avoid habits unweave faster.

This reflects the real behavioral pattern: consecutive repetition strengthens automaticity faster than interrupted repetition covering the same total number of days.

Returns: `double` — the final adjusted percent.

---

### `updateLiveWeekDelta(definitionId, date)`

Updates the live `CommitmentWeeklyProgress` record for the current week.

- Reads current `garmentCompletionPercent`
- Reads `garmentPercentAtWeekStart` stored when the week began
- Calculates delta: current minus week start
- Saves updated live record via `CommitmentRepository.saveWeeklyProgress()`
- If today is Monday and no live record exists → creates a new one, stores current percent as week start baseline

---

### `sealCompletedWeek(definitionId, weekStart)`

Called on Sunday night after `calculateAndSaveDailyUpdate` completes.

- Sets `isCurrentWeek: false` on the live week record
- Creates a new live week record for the coming Monday with `isCurrentWeek: true`

Called by: midnight background scheduler action, Sunday only.

---

## Configurable Constants

All constants are defined in one place inside `DefaultGarmentCalculationService`. Changing any of them changes the behavior for all commitments — no per-commitment configuration.

```
dailyFullContribution: 1.5       // % gained on a fully kept day
dailyBreachPenalty: 2.0          // % lost on an avoid habit breach
decayAmount: 1.0                 // % lost/gained per day when decay is active
streakBonusThreshold: 3          // minimum streak days to activate bonus
streakBonusMultiplier: 1.2       // performance multiplier during active streak
```

These values are placeholders. They will be tuned based on real user data after launch. The abstraction exists precisely so this tuning requires changing one file, not touching anything else.

---

## Avoid Habit Direction

All calculations work identically for both commitment types — the direction is reversed.

|Event|Do habit|Avoid habit|
|---|---|---|
|Successful day|percent increases|percent decreases|
|Failure / breach|percent unchanged (decay may apply)|percent increases|
|Decay active|percent decreases|percent increases|
|Streak bonus|increases faster|decreases faster|
|Starting value|0.0%|100.0%|
|Goal|reach 100.0%|reach 0.0%|

---

## Rules

- Never reads `progressPercent` from instances directly — reads the instance result (kept/missed/percent) via `CommitmentService.getTodayInstances()`
- Never writes to anything except `CommitmentDefinition.garmentCompletionPercent`, `CommitmentDefinition.consecutiveFailDaysForDecay`, `CommitmentDefinition.lastGarmentUpdateDate`, and `CommitmentWeeklyProgress` records
- All sub-functions are pure — no side effects, fully testable without mocks
- Frozen and completed commitments are skipped entirely — no garment update, no weekly record
- Idempotent — `lastGarmentUpdateDate` prevents double-application
- The interface is the contract — the constants and formulas inside the implementation are not

---

## Dependencies

- `CommitmentService.getTodayInstances()` — reads today's instance result
- `CommitmentService.saveDefinitionState()` — writes garment fields (not the user-edit path — see architecture notes)
- `CommitmentRepository.saveWeeklyProgress()` — writes weekly delta records