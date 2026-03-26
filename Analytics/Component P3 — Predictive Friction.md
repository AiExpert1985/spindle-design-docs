**File Name**: component_predictive_friction **Feature**: Analytics **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** warns the user before a failure happens. Detects when current conditions match historical failure patterns and sends a timely nudge while there is still time to act. Delivered as a notification — no UI surface.

Standalone — can be added or removed without touching any other feature or component. Requires 2–3 weeks of history to produce meaningful signals.

---

## How It Works

`PredictiveFrictionService` runs on every `ShortIntervalTickEvent` during waking hours. For each active commitment, it compares current conditions against historical failure patterns from `AnalyticsService`. If two or more risk signals align, it sends a notification via `NotificationService`.

A single signal is never enough — at least two must align to avoid false positives.

---

## Risk Signals

|Signal|Example|
|---|---|
|Time and day matches historical failure pattern|Friday afternoon — 70% of failures happen here|
|Low progress with little window time left|Final 25% of window, under 30% logged|
|Avoid commitment approaching limit|At 1 of 1 allowed coffee with 2 hours left|
|Context tag matches historical failure tag|Tagged "tired" today — 80% of failures tagged "tired"|

---

## Notification Examples

```
"You usually exceed your coffee limit on afternoons like this.
2 hours left. You're at 1 of 1. Stay strong."

"Walk window closes in 90 min.
You haven't logged — and Fridays are your toughest day."
```

---

## Rules

- At least 2 risk signals must align before an alert fires
- Maximum 1 alert per commitment per day — idempotency via stable notification ID incorporating `definitionId` + date, enforced by `NotificationService`
- Never fires if the commitment has already been breached today
- Never fires for frozen or completed commitments
- Exits silently for commitments with fewer than 2–3 weeks of history
- Respects global notification settings from `UserCoreService`

---

## Dependencies

- EventBus — subscribes to `ShortIntervalTickEvent`
- `CommitmentIdentityService.getInstances(from, to, definitionId)` — current instance state
- `AnalyticsService` — historical failure patterns
- `NotificationService` — schedules the alert notification
- `UserCoreService.getProfile()` — notification preference check