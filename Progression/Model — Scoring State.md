**File Name**: model_scoring_state **Feature**: Progression (internal) **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** stores operational state for `ScoringService`. Separates scoring mechanics from the user's progression profile — `ProgressionProfile` describes what the user _has achieved_, `ScoringState` describes what the scoring logic _last did_.

---

## Fields

```
ScoringState
  lastBonusAwardedMonth: DateTime?   // first day of the month of the last bonus award
                                     // null means no bonus ever awarded
  updatedAt: DateTime
```

- **lastBonusAwardedMonth** — enforces the one-bonus-per-month cap. `ScoringService` reads this before evaluating bonus triggers. Written by `ScoringService` when a bonus is awarded. Comparison is month + year only, not exact datetime.

---

## Rules

- One record per user — created on first bonus award
- Written only by `ScoringService`
- Read only by `ScoringService` — never exposed to `ProgressionService` or any external feature