**File Name**: service_analytics **Feature**: Analytics **Phase**: 2 (computeDayFacts) · 3 (all other functions) **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** computes structured facts about commitment performance. Pure calculations — no writes, no repository, no persistence. Reads from Activity, Performance, and Commitment services. Results returned to callers and never stored.

---

## Data Access

|Feature|Function|Used for|
|---|---|---|
|Activity|`ActivityService.getEntriesForPeriod(id, from, to)`|Log entries and notes|
|Activity|`ActivityService.getEntriesForDay(date)`|Day snapshot logs|
|Performance|`PerformanceService.getPerformanceForPeriod(from, to, definitionId?)`|Period scores and trends|
|Performance|`PerformanceService.getDayScore(date)`|Day score|
|Commitment|`CommitmentIdentityService.getInstances(from, to, definitionId?)`|Instance timing, results, and commitment names|

Commitment names read from instance `name` snapshot field — no definition lookup needed.

---

## Note Structure

When `includeNotes: true`, log notes are structured and included in the payload.

```
LogNote
  definitionId: String
  commitmentName: String
  date: DateTime
  logValue: double
  note: String?
  contextTags: List<String>   // empty list until context tags feature ships (Phase 2)
```

---

## Core Functions

### `computeCommitmentFacts(definitionId, from, to, includeNotes)` → CommitmentFactsPayload — Phase 3

- Kept percentage over the period
- Failure rate by hour of day and day of week
- Top context tags on failures and successes (Phase 2+ data — empty until context tags ship)
- Trend direction vs previous equivalent period
- Weekly performance history

### `computeWeeklyFacts(from, to, includeNotes)` → WeeklyFactsPayload — Phase 3

- Overall period score
- Per-commitment trend — improving / declining / steady
- Strongest and weakest commitment
- Best and worst day of week
- Cross-commitment patterns where visible
- Comparison to previous equivalent period

### `computeDayFacts(date)` → DayFactsPayload — Phase 2

Snapshot of one day. Notes not included.

- Day score
- Which commitments were kept and missed
- Most notable pattern visible that day

---

## Rules

- Pure computation — no writes, no repository, no persistence
- Reads from Activity, Performance, and Commitment services only — never their repositories
- All functions exclude frozen instances by default
- Results not cached — callers request fresh computation each time
- Commitment names sourced from instance `name` field — no definition lookups

---

## Dependencies

- `ActivityService` — `getEntriesForPeriod()`, `getEntriesForDay()`
- `PerformanceService` — `getPerformanceForPeriod()`, `getDayScore()`
- `CommitmentIdentityService` — `getInstances()`