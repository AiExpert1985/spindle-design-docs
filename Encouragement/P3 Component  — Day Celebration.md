**File Name**: component_day_celebration **Feature**: Encouragement **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** celebrates the user's day with a brief full-screen ephemeral moment, then navigates to the Dashboard. The Encouragement feature owns the trigger logic and story selection. This component is the visual surface only.

Expected placement: overlaid on the current screen when `DayCelebrationSignal` is emitted. Auto-dismisses and navigates to the Dashboard.

---

## When It Fires

- Day score ≥ `UserDefaultPreferences.celebrationThreshold` (default 60%)
- At most once per day — enforced by `TickGuard` in `EncouragementService`
- Silence below threshold — no celebration, no message, no notification

---

## Two Entry Paths

**Path A — In-app:** `EncouragementService` detects all commitment windows for today are closed via `PerformanceUpdatedEvent(isClosed: true)`. If day score meets threshold → emits `DayCelebrationSignal`. Presentation layer shows the overlay immediately.

**Path B — Notification:** `EncouragementService` fires on `LongIntervalTickEvent` near the configured celebration time (default 9pm). Checks day score, selects story, emits `DayCelebrationSignal`. Also calls `NotificationService.send()` directly with the celebration notification.

Notification examples:

```
"Your garment grew stronger today. 83%"
"After a tough few days, you're back at 71%."
"3km of your 5km walk. Partial progress moves you forward."
```

---

## Ephemeral Screen (~3 seconds)

Full screen, dark overlay. Auto-dismisses and navigates to Dashboard.

```
┌─────────────────────────────────────────┐
│                                         │
│               🎯  83%                   │
│                                         │
│   Your garment grew stronger today.     │
│   After a tough few days, you're back.  │
│                                         │
└─────────────────────────────────────────┘
```

- Score and story animate in on appear
- Auto-dismisses after ~3 seconds → navigates to Dashboard
- Tap anywhere to dismiss early → navigates to Dashboard
- No close button — auto-dismiss always handles it

---

## Score to Short Message

|Score|Message|
|---|---|
|60–69%|"Good effort."|
|70–89%|"Thread woven."|
|≥ 90%|"Excellent."|

---

## Rules

- Purely presentational — receives `DayCelebrationSignal` from the Riverpod provider
- Story text comes from the signal — this component renders, never selects
- Uses garment language — "garment grew stronger", not "rope grew stronger"
- Never fires on bad days — threshold check owned by `EncouragementService`

---

## Dependencies

- Riverpod provider — watches `DayCelebrationSignal` from `EncouragementService`
- Navigation — dismissal navigates to Dashboard

---

## Later Improvements

**Share button.** A share option on the celebration screen allowing the user to share today's result as an image card. Requires a `ShareService` component — not yet designed.

**Dashboard score ring sweep.** After dismissal, the Dashboard score ring sweeps from zero to today's score — continuing the energy of the celebration. Depends on the Dashboard card interaction being designed (Phase 2).

**Configurable threshold and time.** The celebration threshold and notification time are already in `UserDefaultPreferences` — the Settings screen UI to configure them is a Phase 2 addition.