**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Analytics **Phase**: 2

**Purpose:** computes statistical facts about commitment performance for display components and AI analysis. Pure calculations — no AI, no writes, no external calls. Reads from Activity and Performance features. Available to all tiers.

---

## Feature Boundary

Analytics is its own feature. It has no repository and no persistent model — it is a pure calculation layer. It reads from Activity and Performance features, never from Commitment directly.

```
Commitment
    ↓
Activity        ← Analytics reads log entries here
    ↓
Performance     ← Analytics reads daily entries here
    ↓
Analytics       ← pure computation, no storage
    ↓
AI Insights     ← consumes analytics payloads
Encouragement   ← consumes day facts
```

---

## Data Access

**From Activity feature:**

- `ActivityService.getEntriesForPeriod(from, to, definitionId?, includeFrozen?)` — log entries with notes and tags

**From Performance feature:**

- `PerformanceAccountingService.getCommitmentWeeklyDeltas(definitionId)` — weekly performance history
- `PerformanceAccountingService.getLiveWeekDelta(definitionId)` — current week progress

**From Commitment feature:**

- `CommitmentService.getInstancesForPeriod(from, to, definitionId?, includeFrozen?)` — instance results and timing

Dependency direction: Analytics → Activity, Performance, Commitment. One-way. None of those features call Analytics.

---

## Output Variants

Every function produces one payload. The caller controls whether notes are included:

- `includeNotes: false` — stats only. Fast. Used by display components.
- `includeNotes: true` — stats + structured notes. Used by AI Insights service.

---

## Note Structure

When `includeNotes: true`:

**Instance notes** — why an overall instance went well or poorly:

```
instanceNotes: [{
  commitmentId, commitmentName,
  date, dayOfWeek,
  outcome: kept | missed,
  score: double,
  note: String,
  tags: List<String>
}]
```

**Log notes** — why a specific entry happened:

```
logNotes: [{
  commitmentId, commitmentName,
  date, time,
  logValue: double,
  note: String,
  tags: List<String>
}]
```

---

## Core Functions

### `computeCommitmentFacts(id, from, to, includeNotes)`

Computes all stats for one commitment over a period.

Output includes:

- Kept % over the period
- Failure rate by hour of day and day of week
- Top context tags on failures and successes
- Energy level correlation with outcomes
- Trend direction vs previous equivalent period
- Weekly delta history from Performance

Used by: micro-insight component, predictive friction, commitment performance summary.

---

### `computeWeeklyFacts(from, to, includeNotes)`

Computes stats across all active commitments for a period.

Output includes:

- Overall period score
- Per-commitment trend — improving / declining / steady
- Strongest and weakest commitment
- Best and worst day of week
- Cross-commitment correlations where visible
- Comparison to previous equivalent period

Used by: weekly summary component (quick summary and deep report).

---

### `computeDayFacts(date)`

Snapshot of one day across all active commitments.

Output includes:

- Day score
- Which commitments kept and missed
- Most interesting pattern visible that day

Notes not included — used by Encouragement for day celebration story selection.

Used by: `EncouragementService` (day celebration).

---

## Rules

- Pure computation — no writes, no repository, no persistence
- Reads from Activity, Performance, and Commitment services only — never their repositories
- All functions exclude frozen instances unless `includeFrozen: true` explicitly passed
- Results are not stored — AI Insights handles its own storage
- No UI of its own — results consumed by display components and AI Insights

---

## Dependencies

- `ActivityService` — log entries and notes
- `PerformanceAccountingService` — weekly delta history
- `CommitmentService` — instance timing and status data