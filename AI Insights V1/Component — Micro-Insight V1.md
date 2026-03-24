**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: AI Insights **Phase**: 3 **Availability:** Pro / Premium only. **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** delivers a genuinely useful AI analysis of one commitment — deep reasoning about why the pattern exists and concrete recommendations to improve. Appears when the user notices a problem.

---

## Where It Appears

Shown on the commitment detail screen when a commitment's current score is below 50%.

```
Coffee ≤ 1    33%  ⚠

Most failures: 4–6 PM · Worst day: Friday · Top tag: 'tired'
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

**Step 2 — AI reasoning (on tap):** User taps button → full payload assembled → sent to `AIInsightService.generateMicroInsight()` → reasoning appears.

If a previous insight exists in storage, it is shown immediately with a "Regenerate" option.

---

## Storage Behavior

One record saved per commitment — always overwrites previous.

```
This insight will be replaced when you regenerate.
[ Save as PDF ]  [ Share ]
```

---

## Service Logic

- Load stats → `AnalyticsService.computeCommitmentFacts(id, last 30 days, includeNotes: false)`
- Check for existing insight → `AIInsightRepository.getLastInsight(micro_insight, commitmentId)`
- On tap → `AnalyticsService.computeCommitmentFacts(id, last 30 days, includeNotes: true)`
- Generate → `AIInsightService.generateMicroInsight(factsPayload)`
- Save → `AIInsightRepository.saveInsight(record)` — overwrites previous

---

## Dependencies

- `AnalyticsService` — stats and notes payload
- `AIInsightService` — AI reasoning
- `AIInsightRepository` — last insight storage
- `UserService.canAccessFeature()` — tier gate