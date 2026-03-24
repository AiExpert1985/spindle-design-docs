**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Encouragement **Phase**: 2 **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** celebrates the user's day with a brief full-screen ephemeral moment, then navigates to the dashboard. The Encouragement feature owns the trigger logic and story selection. The presentation layer owns the screen.

---

## When It Fires

- Day score ≥ 60% (configurable in settings, default 60%)
- Silence below threshold — no celebration on bad days
- At most once per day

---

## Two Entry Paths

**Path A — In-app:** `EncouragementService` detects all commitment windows for today are closed via `PerformanceEntryWrittenEvent`. If day score meets threshold → emits `DayCelebrationSignal`. Presentation layer shows ephemeral screen immediately.

**Path B — Notification:** Background scheduler fires `MidnightEvent` at configured time (default 9pm). `EncouragementService` subscribes, checks day score, selects story, sends thin notification via event → `NotificationService` subscribes to `DayCelebrationSignal` and sends the notification.

```
"Your rope grew stronger today. 83%"
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
│   Your rope grew stronger today.        │
│   After a tough few days, you're back.  │
│                                         │
│         [ Share today's result ]        │
│                                         │
└─────────────────────────────────────────┘
```

- Score and story animate in
- Auto-dismisses after ~3 seconds → navigates to Dashboard
- Tap anywhere to dismiss early → navigates to Dashboard
- Share button → Template A via `ShareService` (does not dismiss)
- No close button needed — auto-dismisses always

**After dismissal:** Dashboard opens and score ring sweeps to today's score.

---

## Score to Short Message

|Score|Message|
|---|---|
|60–69%|"Good effort."|
|70–89%|"Thread woven."|
|≥ 90%|"Excellent."|

---

## Story Selection

Seven story types selected by priority using `AnalyticsService.computeDayFacts(today)`. Same story used in notification text and ephemeral screen.

|Priority|Type|Example|
|---|---|---|
|1|Comeback|"After a tough few days, you're back at 71%."|
|2|Improvement over yesterday|"Up from 61% to 74%. Moving forward."|
|3|Commitment spotlight|"You walked every day this week. That's becoming part of who you are."|
|4|Consistency (3+ weeks)|"Three weeks of quiet discipline. That's the hardest kind."|
|5|Partial progress|"3km of your 5km walk. Partial progress moves you forward."|
|6|Overall score (fallback)|"83% today. Your rope grew stronger."|
|7|Slight setback (still ≥ threshold)|"Slightly below yesterday, but 65% still counts."|

Type 6 always applies as fallback. Never repeats the same type two consecutive days — tracked via `UserService.recordEncouragementSent(type)`.

---

## Settings

|Setting|Default|Configurable|
|---|---|---|
|Threshold|60%|Yes — Phase 2|
|Notification time|9pm|Yes — Phase 2|
|Enabled|true|Yes — Phase 2|

---

## Dependencies

- `EncouragementService` — owns trigger logic, story selection, signal emission
- `AnalyticsService` — day facts and score
- `UserService` — threshold setting, last story type deduplication
- `ShareService` — share button
- Presentation layer — watches `DayCelebrationSignal` Riverpod provider