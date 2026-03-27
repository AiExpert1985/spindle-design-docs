**File Name**: service_analytics **Feature**: Analytics **Phase**: 2 (computeDayFacts) · Phase 3 (all other functions) **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** computes statistical facts about commitment performance for display components and AI analysis. Pure calculations — no writes, no repository, no persistence. Reads from Activity, Performance, and Commitment features. Available to all tiers.

---

## Design

Analytics has no repository and no model — it is a pure computation layer. Results are returned to callers and never stored. AI Insights handles its own storage of generated insights. This keeps the Analytics feature lightweight and testable — every function is a pure computation given its inputs.

---

## Data Access

|Feature|Function|Used for|
|---|---|---|
|Activity|`ActivityService.getEntriesForPeriod(id, from, to)`|Log entries and notes|
|Activity|`ActivityService.getEntriesForDay(date)`|Day snapshot logs|
|Performance|`PerformanceService.getPerformanceForPeriod(from, to, definitionId?)`|Period scores and trends|
|Performance|`PerformanceService.getDayScore(date)`|Day score|
|Commitment|`CommitmentIdentityService.getInstances(from, to, definitionId?)`|Instance timing, results, and commitment names|

Analytics never accesses any repository directly — always through the owning feature's service.

Commitment names are read from the instance's `name` field — no `CommitmentService.getDefinition()` call needed. The instance already carries the commitment name as a snapshot field.

---

## Output Variants

Functions that accept `includeNotes` produce two variants:

- `includeNotes: false` — stats only. Fast. Used by display components.
- `includeNotes: true` — stats plus structured notes. Used by AI Insights service.

---

## Note Structure

When `includeNotes: true`, notes from `LogEntry` are structured and included in the payload.

```
LogNote
  definitionId: String
  commitmentName: String
  date: DateTime
  logValue: double
  note: String?
  contextTags: List<String>   // Phase 2+ — empty list until context tags feature ships
```

`contextTags` is included in the struct from the start — it is an empty list until the context tags feature is implemented in Phase 2. AI Insights is Phase 3, so context tags will always be available when `includeNotes: true` is used.

---

## Core Functions

### `computeCommitmentFacts(definitionId, from, to, includeNotes) → CommitmentFactsPayload` — Phase 3

Computes all stats for one commitment over a period.

Output includes:

- Kept percentage over the period
- Failure rate by hour of day and day of week — derived from `LogEntry.loggedAt` timestamps
- Top context tags on failures and successes (Phase 2+ data — empty until context tags ship)
- Trend direction vs previous equivalent period
- Weekly performance history

---

### `computeWeeklyFacts(from, to, includeNotes) → WeeklyFactsPayload` — Phase 3

Computes stats across all active commitments for a period.

Output includes:

- Overall period score
- Per-commitment trend — improving / declining / steady
- Strongest and weakest commitment
- Best and worst day of week
- Cross-commitment patterns where visible
- Comparison to previous equivalent period

---

### `computeDayFacts(date) → DayFactsPayload` — Phase 2

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
- Commitment names sourced from instance `name` field — no definition lookups

---

## Dependencies

- `ActivityService` — log entries and notes via `getEntriesForPeriod()`, `getEntriesForDay()`
- `PerformanceService` — period scores, day scores
- `CommitmentIdentityService` — instance timing, results, and commitment names

---

## Later Improvements

**Context tag analysis.** Top context tags on failures and successes. Requires context tags feature (Phase 2) to be fully populated before patterns are meaningful.

**Energy level correlation.** Correlating outcomes with user-reported energy level (1–5 scale). Requires `energyLevel` field to be added to `LogEntry` — see `component_context_tags` later improvements.