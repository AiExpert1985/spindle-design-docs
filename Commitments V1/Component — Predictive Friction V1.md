**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Commitment **Phase**: 3 **Availability:** all tiers — requires 2–3 weeks of history. **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** warns the user before a failure happens. Detects when current conditions match historical failure patterns and sends a timely nudge while there is still time to act.

---

## How It Works

Subscribes to `WindowCheckEvent` (fired every 15 minutes during waking hours 7am–10pm). For each active commitment, compares current conditions against historical failure patterns. If enough risk signals align → publishes `PredictiveFrictionAlertEvent`. `NotificationService` subscribes and sends the warning.

A single signal is not enough — at least 2 must align to avoid false positives.

---

## Risk Signals

|Signal|Example|
|---|---|
|Time/day matches historical failure pattern|Friday afternoon — 70% of failures happen here|
|Low progress with little window time left|Final 25% of window, under 30% logged|
|Avoid commitment approaching limit|At 1 of 1 allowed coffee with 2 hours left|
|Context tag matches historical failure tag|Tagged "tired" today — 80% of failures tagged "tired"|

---

## Example Notifications

```
"You usually exceed your coffee limit on afternoons like this.
2 hours left. You're at 1 of 1. Stay strong."

"Walk window closes in 90 min.
You haven't logged — and Fridays are your toughest day."
```

---

## Service Logic

`PredictiveFrictionService` subscribes to `WindowCheckEvent`:

- For each active commitment:
    - Fetch historical patterns → `AnalyticsService.computeCommitmentFacts(id, last 60 days, includeNotes: false)`
    - Fetch current instance → `CommitmentService.getInstancesForPeriod(today, today, id)`
    - Evaluate risk signals against current conditions
    - If ≥ 2 signals align → publish `PredictiveFrictionAlertEvent(instanceId, message)`
    - Record alert date via `CommitmentService.saveDefinitionState()` to prevent repeat alerts today

`NotificationService` subscribes to `PredictiveFrictionAlertEvent` → sends notification.

---

## Rules

- At least 2 risk signals must align
- Maximum 1 alert per commitment per day — tracked via `lastFrictionAlertDate`
- Never fires if commitment already breached
- Never fires for frozen commitments
- Requires 2–3 weeks of history — exits silently for new commitments
- Respects global notification settings from `UserService`

---

## Data Model Addition

```
lastFrictionAlertDate: DateTime?    // on CommitmentDefinition — null if never alerted
```

---

## Dependencies

- `EventBus` — subscribes to `WindowCheckEvent`, publishes `PredictiveFrictionAlertEvent`
- `AnalyticsService` — historical failure patterns
- `CommitmentService` — current instance state, saves alert date
- `NotificationService` — subscribes to alert event, sends notification