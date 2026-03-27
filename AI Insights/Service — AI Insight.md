**File Name**: service_ai_insight **Feature**: AI Insights **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

**Availability:** Pro / Premium only.

---

**Purpose:** takes pre-computed facts and notes from `AnalyticsService` and produces genuine AI reasoning — causes, patterns, and actionable recommendations. Owns insight storage and all rate limit enforcement. Abstract interface — the rest of the app never knows which AI provider is active.

---

## Abstraction

```dart
abstract class AIInsightService {
  Future<Result<AIInsightRecord>> generateMicroInsight(String definitionId);
  Future<Result<AIInsightRecord>> generateQuickSummary(DateTime weekStart);
  Future<Result<AIInsightRecord>> generateDeepReport(DateTime from, DateTime to);
  Future<AIInsightRecord?> getLastInsight(InsightType type, {String? definitionId});
}
```

```
AIInsightService (abstract)
  → AnthropicInsightService    (current implementation)
  → (future: OpenAI, Gemini, others)
```

Components call `AIInsightService` only. Swapping the provider means adding one new implementation class — nothing else changes.

---

## Functions

### `generateMicroInsight(definitionId) → Result<AIInsightRecord>`

1. Check free-tier quota via `UserSettingsService.checkAndDecrementInsightQuota()`. Returns `Failure` if quota exhausted — the presentation layer shows the upgrade prompt.
2. Fetch facts via `AnalyticsService.computeCommitmentFacts(definitionId, last 30 days, includeNotes: true)`.
3. Format payload into prompt internally.
4. Call AI provider. Returns `Failure` on network or API error.
5. Save result via `AIInsightRepository.saveInsight(record)`.
6. Return `Success(record)`.

Pro/Premium users have no quota. Free users have a configurable weekly limit from `AppConfig.freeInsightsPerWeek`.

### `generateQuickSummary(weekStart) → Result<AIInsightRecord>`

Pro/Premium only — quota never applies.

1. Fetch facts via `AnalyticsService.computeWeeklyFacts(weekStart, weekEnd, includeNotes: false)`.
2. Format payload into prompt.
3. Call AI provider. Returns `Failure` on error.
4. Save result via `AIInsightRepository.saveInsight(record)`.
5. Return `Success(record)`.

### `generateDeepReport(from, to) → Result<AIInsightRecord>`

Pro/Premium only. Rate-limited to once per 7 days.

1. Check `AIInsightRepository.getLastInsight(deepReport)`. If `generatedAt` is within 7 days, return `Failure(AppError(type: validation, message: 'Deep report already generated this week'))`. The presentation layer shows the user when the next report is available.
2. Fetch facts via `AnalyticsService.computeWeeklyFacts(from, to, includeNotes: true)`.
3. Format payload into prompt.
4. Call AI provider. Returns `Failure` on error.
5. Save result via `AIInsightRepository.saveInsight(record)`.
6. Return `Success(record)`.

### `getLastInsight(type, definitionId?) → AIInsightRecord?`

Returns the last stored insight of the given type, or null. Called by components to show a cached insight before offering regeneration.

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

---

## Rules

- Input is always pre-computed facts from `AnalyticsService` — never raw instances or log entries
- AI provider receives no user identifiers — only behavioral facts and anonymized notes
- All rate limit and quota enforcement happens here — components never enforce limits themselves
- The service formats the payload into a prompt internally — components never construct prompts
- All functions return `Result<T>` — failures (quota exhausted, rate limited, API error) surface to the presentation layer
- Written records go through `AIInsightRepository` — no other feature writes insight records

---

## Dependencies

- `AIInsightRepository` — reads and writes insight records
- `AnalyticsService` — computes facts payload for all three insight types
- `UserSettingsService.checkAndDecrementInsightQuota()` — free-tier quota check (micro-insight only)
- `AppConfig` — `freeInsightsPerWeek`, deep report rate limit window