
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 2

**Standalone:** yes — can be added to any screen showing one commitment's performance.

**Purpose:** shows a quick structured performance summary for one commitment over the last 30 days. Richer than a single score, simpler than full AI analysis. No AI involved — pure calculation.

---

## UI

```
Last 30 days

74%  overall        ↑ improving vs previous 30 days
Best day: Tuesday   Toughest: Friday
Most consistent: mornings   Most slips: afternoons
```

- Overall score for the period
- Trend arrow vs previous equivalent period (improving / declining / steady)
- Best and toughest day of week
- Timing pattern if detectable (mornings / afternoons / evenings)

Tap anywhere on this section → expands to show more detail inline. No navigation needed.

---

## What It Shows

|Stat|Meaning|
|---|---|
|Overall score|Average `progressPercent` for last 30 days|
|Trend|Compared to previous 30 days — up/down/steady|
|Best day|Day of week with highest kept %|
|Toughest day|Day of week with lowest kept %|
|Timing pattern|Hour cluster where most successes or failures occur|

---

## Service Logic

Calls `CommitmentService.getCommitmentPerformance(id, last 30 days)` which returns the structured performance facts. No AI, no pre-computation layer — pure calculation in the service.

---

## Dependencies

- `CommitmentService.getCommitmentPerformance(id, from, to)` — structured performance facts