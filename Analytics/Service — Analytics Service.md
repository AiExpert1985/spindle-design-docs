**File Name**: service_analytics **Feature**: Analytics **Phase**: 2 (computeDayFacts) · Phase 3 (all other functions) **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** computes statistical facts about commitment performance for display components and AI analysis. Pure calculations — no writes, no repository, no persistence. Reads from Activity, Performance, and Commitment features. Available to all tiers.

---

## Design

Analytics has no repository and no model — it is a pure computation layer. Results are returned to callers and never stored. AI Insights handles its own storage of generated insights. This keeps the Analytics feature lightweight and testable — every function is a pure computation given its inputs.

---

## Data Access

|Feature|Function|Used for|
|---|---|---|
|Activity|`ActivityService.getEntriesForCommitment(id, from, to)`|Log entries, notes, tags|
|Activity|`ActivityService.getEntriesForDay(date)`|Day snapshot logs|
|Performance|`PerformanceService.getPerformanceForPeriod(from, to, definitionId?)`|Period scores and trends|
|Performance|`PerformanceService.getDayScore(date)`|Day score|
|Commitment|`CommitmentIdentityService.getInstances(from, to, definitionId?)`|Instance timing and results|

Analytics never accesses any repository directly — always through the owning feature's service.

---

## Output Variants

Functions that accept `includeNotes` produce two variants:

- `includeNotes: false` — stats only. Fast. Used by display components.
- `includeNotes: true` — stats plus structured notes. Used by AI Insights service.

---

## Note Structure

When `includeNotes: true`, notes from `LogEntry` are structured and included in the payload.

**Log notes** — why a specific entry happened:

```
LogNote
  definitionId: String
  commitmentName: String
  date: DateTime
  logValue: double
  note: String?
  contextTags: List<String>
```

---

## Core Functions

### `computeCommitmentFacts(definitionId, from, to, includeNotes)` — Phase 3

Computes all stats for one commitment over a period.

Output includes:

- Kept percentage over the period
- Failure rate by hour of day and day of week — derived from `LogEntry.loggedAt` timestamps
- Top context tags on failures and successes
- Trend direction vs previous equivalent period
- Weekly performance history

Used by: `component_commitment_performance_summary`, `component_predictive_friction`, `service_ai_insight`.

---

### `computeWeeklyFacts(from, to, includeNotes)` — Phase 3

Computes stats across all active commitments for a period.

Output includes:

- Overall period score
- Per-commitment trend — improving / declining / steady
- Strongest and weakest commitment
- Best and worst day of week
- Cross-commitment patterns where visible
- Comparison to previous equivalent period

Used by: weekly summary component (quick summary and deep report).

---

### `computeDayFacts(date)` — Phase 2

Snapshot of one day across all active commitments. Notes not included — used for celebration story selection, not AI analysis.

Output includes:

- Day score
- Which commitments were kept and missed
- Most notable pattern visible that day

Used by: `EncouragementService` — day celebration story selection.

---

## Rules

- Pure computation — no writes, no repository, no persistence
- Reads from Activity, Performance, and Commitment services only — never their repositories
- All functions exclude frozen instances by default
- Results are not cached — callers request fresh computation each time
- No UI of its own — results consumed by display components and AI Insights

---

## Dependencies

- `ActivityService` — log entries, notes, context tags
- `PerformanceService` — period scores, day scores
- `CommitmentIdentityService` — instance timing and results

---

## Later Improvements

**Energy level correlation.** Correlating outcomes with user-reported energy level (1–5 scale). Requires `energyLevel` field to be added to `LogEntry` — see `component_context_tags` later improvements.