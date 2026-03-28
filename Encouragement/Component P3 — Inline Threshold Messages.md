**File Name**: component_inline_threshold_messages **Feature**: Encouragement **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** short contextual messages that appear at score milestones during a logging session. Reinforces progress without requiring the user to look away from what they just logged.

Expected placement: inline below the commitment card that was just logged, on the Dashboard. Depends on the Dashboard card interaction being designed.

---

## When They Appear

Shown inline after a log, at meaningful score crossings only — never on every log. Fades after 2 seconds.

|Trigger|Message|
|---|---|
|First log of the day on any commitment|"Good start."|
|Commitment `livePerformance` crosses 50%|"Halfway there."|
|Commitment `livePerformance` reaches 100%|"Commitment kept."|
|Overall day score crosses 80%|"You're on track."|

---

## Rules

- Purely presentational — no service calls, no event subscriptions, no storage
- Observes `ThresholdMessageSignal` from `EncouragementService` via Riverpod provider
- Detection logic lives in `EncouragementService` — this component only renders what it receives
- Fades after 2 seconds — never blocks interaction
- Never shown on misses, breaches, or slips — positive moments only
- One message per crossing per session — deduplication owned by `EncouragementService`
- Silent if animation system is under load — drop rather than delay. See `ux_principles` rule 5

---

## Data Sources

|Data|Source|
|---|---|
|Threshold message text|`ThresholdMessageSignal` via Riverpod provider (from `EncouragementService`)|

---

## Dependencies

- Riverpod provider — observes `ThresholdMessageSignal` from `EncouragementService`