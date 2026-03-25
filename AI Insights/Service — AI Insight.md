
**Created**: 15-Mar-2026
**Modified**: -
**Feature:** ai insights 
**Phase:** 3

**Availability:** Pro / Premium only.

**Purpose:** takes pre-computed facts and notes from the analysis service and produces genuine AI reasoning — causes, patterns, and actionable recommendations. Abstract interface — the rest of the app never knows which provider is active.

---

## Abstraction

```
AIInsightService (abstract)
  → AnthropicInsightService    (current implementation)
  → (future: OpenAI, Gemini, others)
```

Components call `AIInsightService` only. Swapping the provider means adding one new implementation class — nothing else changes.

---

## Core Functions

### `generateMicroInsight(commitmentFactsPayload)`

Takes the full `CommitmentFactsPayload` (stats + 30 days of notes) for one commitment and returns a 2–4 sentence reasoning insight.

AI focus: explain _why_ the pattern exists and give one or two specific actionable recommendations. Not a restatement of stats — the user already sees those.

Returns: narrative string. Called by: micro-insight component.

---

### `generateQuickSummary(weekFactsPayload)`

Takes `WeeklyFactsPayload` for the current week (stats only, no notes) and returns a short weekly digest.

AI focus: one observation about the week and one small suggestion. Brief and warm — not a deep analysis.

Returns: narrative string. Called by: weekly summary component (auto Sunday trigger).

---

### `generateDeepReport(weekFactsPayload)`

Takes `WeeklyFactsPayload` for the past 4–8 weeks (stats + full notes) and returns a comprehensive analysis.

AI focus: cross-commitment correlations, root causes from timing and notes, what is genuinely improving vs just a good period, two or three prioritized and specific recommendations.

Returns: narrative string. Called by: weekly summary component (manual trigger only).

---

## Rules

- Input is always a pre-computed payload from `CommitmentAnalyticsService` — never raw instances or log entries
- AI provider receives no user identifiers — only behavioral facts and anonymized notes
- `generateDeepReport` rate-limited to once per week — enforced by the weekly summary component before calling
- All functions are behind the tier gate — free users never reach this service
- The service formats the payload into a prompt internally — components never construct prompts

---

## Prompt Rules

These rules apply to every prompt written inside this service. The AI writes narrative — it never judges the user.

**Tone table:**

|Avoid|Use instead|
|---|---|
|"Why did you fail?"|"What pattern caused this?"|
|"You failed"|"Commitment skipped"|
|"You should do better"|"Here's what the data shows"|
|"You are bad at X"|"X is your toughest pattern right now"|

**Three rules that apply to all prompts:**

- **Analytical, never moral** — describe what happened, never judge the person
- **Specific, never generic** — "you skip walk on Fridays after 5pm" beats "you struggle with consistency"
- **One or two recommendations maximum** — more than two is overwhelming and none get acted on