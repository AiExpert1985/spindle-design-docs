**File Name**: service_ai_insight **Feature**: AI Insights **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

**Availability:** Pro / Premium only.

---

**Purpose:** takes pre-computed facts from `AnalyticsService` and produces AI reasoning — causes, patterns, and actionable recommendations. Owns insight storage and all rate limit enforcement. Abstract interface — the rest of the app never knows which AI provider is active.

---

## Abstraction

```dart
AIInsightService (abstract)
  → AnthropicInsightService    (current implementation)
  → (future: OpenAI, Gemini, others)
```

Components call `AIInsightService` only. Swapping the provider means adding one new implementation class — nothing else changes.

---

## Functions

### `generateMicroInsight(definitionId)` → Result<AIInsightRecord>

1. Check free-tier quota via `UserSettingsService.checkAndDecrementInsightQuota()`. Returns `Failure` if quota exhausted.
2. Fetch `AnalyticsService.computeCommitmentFacts(definitionId, last 30 days, includeNotes: true)`.
3. Format payload into prompt internally.
4. Call AI provider. Returns `Failure` on network or API error.
5. Save via `AIInsightRepository.saveInsight(record)`.
6. Return `Success(record)`.

Pro/Premium users have no quota. Free users have `AppConfig.freeInsightsPerWeek` limit.

### `generateQuickSummary(weekStart)` → Result<AIInsightRecord>

Pro/Premium only.

1. Fetch `AnalyticsService.computeWeeklyFacts(weekStart, weekEnd, includeNotes: false)`.
2. Format payload into prompt.
3. Call AI provider.
4. Save and return.

### `generateDeepReport(from, to)` → Result<AIInsightRecord>

Pro/Premium only. Rate-limited to once per 7 days.

1. Check `AIInsightRepository.getLastInsight(deepReport)`. If within 7 days, return `Failure` with next-available date.
2. Fetch `AnalyticsService.computeWeeklyFacts(from, to, includeNotes: true)`.
3. Format payload into prompt.
4. Call AI provider.
5. Save and return.

### `getLastInsight(type, definitionId?)` → AIInsightRecord?

Returns the last stored insight of the given type, or null. Called by components to show a cached insight before offering regeneration.

---

## Prompt Rules

**Tone:**

|Avoid|Use instead|
|---|---|
|"Why did you fail?"|"What pattern caused this?"|
|"You failed"|"Commitment skipped"|
|"You should do better"|"Here's what the data shows"|
|"You are bad at X"|"X is your toughest pattern right now"|

**Three rules for all prompts:**

- **Analytical, never moral** — describe what happened, never judge the person
- **Specific, never generic** — "you skip walk on Fridays after 5pm" beats "you struggle with consistency"
- **One or two recommendations maximum** — more is overwhelming and none get acted on

---

## Rules

- Input is always pre-computed facts from `AnalyticsService` — never raw instances or log entries
- AI provider receives no user identifiers — behavioral facts and anonymized notes only
- All rate limit and quota enforcement happens here — components never enforce limits
- Service formats the payload into prompts internally — components never construct prompts
- All functions return `Result<T>` — failures surface to the presentation layer
- Written records go through `AIInsightRepository` only

---

## Dependencies

- `AIInsightRepository` — reads and writes insight records
- `AnalyticsService` — computes facts payload for all three insight types
- `UserSettingsService.checkAndDecrementInsightQuota()` — free-tier quota (micro-insight only)
- `AppConfig` — `freeInsightsPerWeek`, deep report rate limit window