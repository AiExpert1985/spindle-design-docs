**File Name**: component_micro_insight **Feature**: AI Insights **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

**Availability:** Pro / Premium only. Free tier users see a quota-limited version — see below. **Standalone:** yes — can be added or removed without touching other components.

---

**Purpose:** delivers a genuinely useful AI analysis of one commitment — deep reasoning about why the pattern exists and concrete recommendations to improve. Appears when the user notices a problem.

---

## Where It Appears

Shown on the commitment detail screen when a commitment's current score is below 50%.

```
Coffee ≤ 1    33%  ⚠

Most failures: 4–6 PM · Worst day: Friday
Past 30 days: 67% failure rate — declining trend

[ Understand why — get AI insight ]
```

The stats line comes from `AnalyticsService.computeCommitmentFacts()` — no AI involved.

---

## What the AI Actually Does

The AI receives 30 days of structured facts. Its job is to reason about causes and recommend actions — not restate the stats the user already sees.

**Example output:**

```
Your coffee failures cluster on Friday afternoons after 5pm,
almost always alongside 'social' tags and notes mentioning
client meetings. This is a social context trigger, not a
willpower problem. Consider a personal rule: one coffee
maximum at social events, or switching to tea after 3pm
on Fridays specifically.
```

---

## Two-Step Reveal

**Step 1 — Instant (no API call):** Stats displayed immediately from `AnalyticsService.computeCommitmentFacts(id, last 30 days, includeNotes: false)`.

**Step 2 — AI reasoning (on tap):** User taps button → `AIInsightService.generateMicroInsight(definitionId)` called → reasoning appears.

If a previous insight exists, it is shown immediately with a "Regenerate" option. Load via `AIInsightService.getLastInsight(microInsight, definitionId)`.

---

## Free Tier Quota

Free users have a weekly micro-insight quota (configured in `AppConfig.freeInsightsPerWeek`). The quota is enforced inside `AIInsightService` — this component only reacts to the result.

When `AIInsightService.generateMicroInsight()` returns a `Failure` due to quota exhaustion, the component shows:

```
You've used your free insights for this week.
Upgrade to Pro for unlimited insights.

[ Upgrade ]   [ Not now ]
```

---

## Storage Behavior

One record saved per commitment — always overwrites previous. `AIInsightService` handles all storage.

```
This insight will be replaced when you regenerate.
[ Save as PDF ]  [ Share ]
```

---

## Error Handling

On `Failure` from `AIInsightService`:

- Quota exhausted → show upgrade prompt
- API error → show: "Couldn't generate insight. Check your connection and try again."

The component never inspects the error type directly — it reads the `AppError.type` field to branch between error messages.

---

## Service Calls

- Load stats → `AnalyticsService.computeCommitmentFacts(id, last 30 days, includeNotes: false)`
- Load cached insight → `AIInsightService.getLastInsight(microInsight, definitionId)`
- Generate → `AIInsightService.generateMicroInsight(definitionId)`

All repository access is handled by `AIInsightService` — this component never calls a repository directly.

---

## Dependencies

- `AnalyticsService` — stats payload for display
- `AIInsightService` — cached insight, generation, storage