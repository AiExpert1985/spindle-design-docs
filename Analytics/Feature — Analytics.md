**File Name**: feature_analytics **Phase**: 2 (computeDayFacts) · 3 (all other functions) **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Analytics is the computation layer that turns raw activity and performance data into structured facts. It produces nothing of its own — it reads from Activity, Performance, and Commitment, computes, and returns. No storage, no events, no side effects. A pure function library that any feature above it can call.

---

## Why It Exists

Performance scores tell you how well a commitment was kept. Analytics tells you why and when — failure patterns by hour and day of week, trends over time, consistency signals, cross-commitment comparisons. This layer exists so that display components and the AI insights feature can receive pre-digested facts rather than pulling raw data and doing their own computation.

Centralizing computation here also means the logic is testable in isolation and consistent across all consumers.

---

## Position in the System

Sits above Activity, Performance, and Commitment. Never accesses their repositories directly — always through their service interfaces. Features that need behavioral facts (display components, AI insights) call down into it. Nothing below it depends on it.

---

## How It Works

Analytics exposes three computation functions. Each accepts a time window and returns a structured payload of facts. Results are never cached — every call is a fresh computation.

**`computeCommitmentFacts(definitionId, from, to, includeNotes)`** — all stats for one commitment over a period. Kept percentage, failure patterns by hour and day of week, trend vs previous equivalent period, weekly performance history.

**`computeWeeklyFacts(from, to, includeNotes)`** — cross-commitment stats for a period. Overall score, per-commitment trend direction, strongest and weakest commitment, best and worst day of week, comparison to previous period.

**`computeDayFacts(date)`** — snapshot of one day across all active commitments. Day score, which commitments were kept and missed, most notable pattern. Used by the Encouragement feature for celebration story selection. Notes not included — this is for behavioral context, not AI analysis.

**The `includeNotes` parameter.** The two Phase 3 functions accept `includeNotes: true/false`. Without notes, the output is stats only — fast, used by display components. With notes, structured log notes are included in the payload — used by AI Insights to give the AI richer context. This one flag covers both use cases without duplicating functions.

Commitment names are read from the instance's `name` snapshot field — no definition lookup needed.

---

## Rules

- Pure computation — no writes, no repository, no persistence
- Reads only through service interfaces — never directly from repositories
- Frozen instances excluded from all computations by default
- Results not cached — fresh computation on every call
- No UI of its own — results consumed by display components and AI Insights
- `computeDayFacts` is Phase 2 — the only function available before Phase 3

---

## Later Improvements

**Context tag analysis.** Top context tags on failures and successes. Requires the context tags feature (Phase 2) to be populated before patterns are meaningful.

**Energy level correlation.** Correlating outcomes with user-reported energy level. Requires `energyLevel` field on `LogEntry` — see `component_context_tags` later improvements.


---

## Related Docs

[[P3 Component — Commitment Performance Summary]]
[[P3 Component — Predictive Friction]]
[[Service — Analytics Service]]