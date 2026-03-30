**File Name**: component_weekly_summary **Feature**: AI Insights **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

**Availability:** Pro / Premium only. **Standalone:** yes — can be added or removed without touching other components.

---

**Purpose:** delivers two levels of weekly AI analysis — a lightweight auto-generated digest every Sunday, and a deep on-demand report when the user wants serious analysis.

---

## Two Report Types

### Quick Summary — auto-generated every Sunday

Analyzes the current week only. Low cost, passive delivery.

**What it covers:** overall week score, per-commitment trend, one observation, one small suggestion.

**Trigger:** `WeekEndedEvent` published by `CommitmentIdentityService`. `WeeklySummaryService` subscribes, calls `AIInsightService.generateQuickSummary(weekStart)`, and on success sends a notification via `NotificationService.push()`.

---

### Deep Report — manually triggered by user

Analyzes past 4–8 weeks including all user notes. Full AI reasoning.

**What it covers:** patterns across all commitments, cross-commitment correlations, root cause analysis, what is genuinely improving, two or three prioritized recommendations, identity statement.

**Trigger:** user taps "Generate deep report." Rate-limited to once per 7 days — enforced by `AIInsightService`, which returns a `Failure` if called within the window.

---

## Storage Behavior

One record saved per type — always overwritten. `AIInsightService` handles all storage.

```
This report will be replaced when you generate a new one.
[ Save as PDF ]  [ Share ]
```

---

## Service Logic

**Quick summary (auto):**

1. `TemporalHelperService.onWeekEnded` fires → `WeeklySummaryService` subscribes
2. Call `AIInsightService.generateQuickSummary(weekStart)`
3. On `Success` → call `NotificationService.push(message, route?)` to notify user
4. On `Failure` → log via `LoggerService`, no notification sent

**Deep report (manual):**

1. User taps → call `AIInsightService.generateDeepReport(from, to)`
2. On `Success` → display report
3. On `Failure(type: validation)` → show: "Your next deep report is available on [date]."
4. On `Failure(type: network / aiApi)` → show: "Couldn't generate report. Check your connection and try again."

All rate limit enforcement is inside `AIInsightService` — this component only reacts to the `Result`.

---

## Error Handling

The component never enforces rate limits or checks storage directly. It calls `AIInsightService` and branches on the `Result`:

- `Success` → display the narrative
- `Failure(validation)` → rate limit message with next available date
- `Failure(network / aiApi)` → retry prompt

---

## Dependencies

- `TemporalHelperService` — subscribes to `onWeekEnded` (for auto quick summary trigger)
- `AIInsightService` — generation, cached insight retrieval, storage
- `NotificationService` — sends summary-ready notification on quick summary success