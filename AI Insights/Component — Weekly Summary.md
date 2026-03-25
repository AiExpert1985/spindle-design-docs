**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: AI Insights **Phase**: 3 **Availability:** Pro / Premium only. **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** delivers two levels of weekly AI analysis — a lightweight auto-generated digest every Sunday, and a deep on-demand report when the user wants serious analysis.

---

## Two Report Types

### Quick Summary — auto-generated every Sunday

Analyzes the current week only. Low cost, passive delivery.

**What it covers:** overall week score, per-commitment trend, one observation, one small suggestion.

**Trigger:** `WeekEndedEvent` published by the scheduler. `WeeklySummaryService` subscribes, generates report, publishes notification via `NotificationService`.

---

### Deep Report — manually triggered by user

Analyzes past 4–8 weeks including all user notes. Full AI reasoning.

**What it covers:** patterns across all commitments, cross-commitment correlations, root cause analysis, what is genuinely improving, two or three prioritized recommendations, identity statement.

**Trigger:** user taps "Generate deep report". Maximum once per week.

---

## Storage Behavior

One record saved per type — always overwritten.

```
This report will be replaced when you generate a new one.
[ Save as PDF ]  [ Share ]
```

---

## Service Logic

**Quick summary (auto):**

- `WeekEndedEvent` fires → `WeeklySummaryService` subscribes
- `AnalyticsService.computeWeeklyFacts(weekStart, includeNotes: false)`
- `AIInsightService.generateQuickSummary(weekFacts)`
- `AIInsightRepository.saveInsight(record)`
- Publishes event → `NotificationService` sends thin notification

**Deep report (manual):**

- User taps → period confirmed
- Check last generation date — block if within last 7 days
- `AnalyticsService.computeWeeklyFacts(from, to, includeNotes: true)`
- `AIInsightService.generateDeepReport(weekFacts)`
- `AIInsightRepository.saveInsight(record)`

---

## Dependencies

- `AnalyticsService` — facts and notes payload
- `AIInsightService` — AI reasoning
- `AIInsightRepository` — last report storage
- `EventBus` — subscribes to `WeekEndedEvent`
- `NotificationService` — subscribes to summary-ready event
- `UserService.canAccessFeature()` — tier gate