**File Name**: component_predictive_friction **Feature**: Analytics **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** warns the user before a failure happens. Detects when current conditions match historical failure patterns and sends a timely nudge while there is still time to act. Delivered as a notification — no UI surface.

Standalone — can be added or removed without touching any other feature or component. Requires 2–3 weeks of history to produce meaningful signals.

---

## How It Works

`PredictiveFrictionService` subscribes to `Heartbeat.shortIntervalTick`. On each tick during waking hours, for each active commitment, it compares current conditions against historical failure patterns from `AnalyticsService`. If two or more risk signals align, it sends a notification via `NotificationService.push()`.

A single signal is never enough — at least two must align to avoid false positives.

---

## Risk Signals

|Signal|Example|
|---|---|
|Time and day matches historical failure pattern|Friday afternoon — 70% of failures happen here|
|Low progress with little window time left|Final 25% of window, under 30% logged|
|Avoid commitment approaching limit|At 1 of 1 allowed coffee with 2 hours left|
|Context tag matches historical failure tag (Phase 2+)|Tagged "tired" today — 80% of failures tagged "tired"|

The context tag signal requires context tags on `LogEntry` (Phase 2). Since Predictive Friction is Phase 3, context tags will be available. If context tags are not yet populated for a commitment, this signal is simply absent — the other three signals still function normally.

---

## Notification Examples

```
"You usually exceed your coffee limit on afternoons like this.
2 hours left. You're at 1 of 1. Stay strong."

"Walk window closes in 90 min.
You haven't logged — and Fridays are your toughest day."
```

---

## Idempotency

Maximum 1 alert per commitment per day. This is tracked in memory — a `Set<String>` keyed by `definitionId + date`, held in `PredictiveFrictionService`. It is never persisted.

This is acceptable: the set resets on app restart, but the tick only fires during an active session. A user who restarts the app mid-day will not see a duplicate alert in practice because the next tick evaluation will find the same conditions but the notification has already been dismissed or acted on.

---

## Rules

- At least 2 risk signals must align before an alert fires
- Maximum 1 alert per commitment per day — in-memory idempotency via `definitionId + date`
- Never fires if the commitment has already been breached or completed today
- Never fires for frozen or completed commitments
- Exits silently for commitments with fewer than 2–3 weeks of history
- Respects global notification settings from `UserCoreService`

---

## Dependencies

- `Heartbeat` — subscribes to `shortIntervalTick`
- `CommitmentIdentityService.getInstances(from, to, definitionId)` — current instance state
- `AnalyticsService` — historical failure patterns
- `NotificationService` — delivers the alert notification
- `UserCoreService.getProfile()` — notification preference check and waking hours
- `TemporalHelperService` — waking hours check