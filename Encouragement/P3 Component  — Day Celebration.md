**File Name**: component_day_celebration **Feature**: Encouragement **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** the visual surface for the day celebration moment. Renders the score, story text, and short score label received from `DayCelebrationSignal`. Purely presentational — never selects stories or checks thresholds.

Expected placement: overlaid on the current screen when `DayCelebrationSignal` is emitted. Auto-dismisses and navigates to the Dashboard.

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

---

## Dependencies

- Riverpod provider — watches `DayCelebrationSignal` from `EncouragementService`
- Navigation — dismissal navigates to Dashboard

---

## Later Improvements

**Share button.** A share option allowing the user to share today's result as an image card. Requires a `ShareService` component — not yet designed.

**Dashboard score ring sweep.** After dismissal, the Dashboard score ring sweeps from zero to today's score. Depends on the Dashboard card interaction being designed (Phase 2).

**Configurable threshold and time.** Already in `UserDefaultPreferences` — the Settings screen UI is a Phase 2 addition.