**File Name**: component_commitment_performance_summary **Feature**: Analytics **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** shows a structured performance summary for one commitment over the last 30 days. Richer than a single score, simpler than full AI analysis. Pure calculation — no AI involved.

Expected placement: Commitment Detail screen, below the commitment facts. Shown once enough history exists (minimum 30 days of instances).

---

## UI

```
Last 30 days

74%  overall        ↑ improving vs previous 30 days
Best day: Tuesday   Toughest: Friday
Most consistent: mornings
```

Tap anywhere → expands inline to show more detail. No navigation.

---

## What It Shows

|Stat|Meaning|
|---|---|
|Overall score|Average `livePerformance` across last 30 closed instances|
|Trend|Compared to previous 30 days — improving / declining / steady|
|Best day|Day of week with highest average performance|
|Toughest day|Day of week with lowest average performance|
|Timing pattern|Hour cluster where most successes or failures occur — shown only when pattern is detectable|

---

## Rules

- Requires minimum 30 closed instances — hidden entirely before that threshold
- Timing pattern shown only when a clear cluster is detectable — never shown as a guess
- Read-only — no interactions except the expand tap

---

## Data Sources

|Data|Source|
|---|---|
|Performance facts|`AnalyticsService` — 30-day per-commitment analysis|

---

## Later Improvements

**Extended period selector.** Allow the user to switch between last 30, 60, and 90 days. Requires no model changes — just a different date range passed to `AnalyticsService`.