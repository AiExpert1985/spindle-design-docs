**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Commitment **Phase**: 1 **Standalone:** yes — can be added or removed without touching other components. **Applies to:** do commitments only. Avoid commitments breach immediately — no check-in.

**Purpose:** when a do commitment window closes with no log, instead of silently marking it missed, the app asks. Treats the user as someone who might have forgotten — not someone who failed.

---

## Check-In Notification

`CommitmentSchedulerService` detects a closed window with no log entry and publishes `WindowClosedWithNoLogEvent`. `NotificationService` subscribes and sends the rich check-in notification.

```
"Morning Walk" window just closed. Did you do it?

[ Yes, I did it ]      → ActivityService.markInstanceKept(instanceId)
[ Give me 15 min ]     → ActivityService.applyGracePeriod(instanceId)
[ Skip — next time ]   → ActivityService.markInstanceMissed(instanceId)
```

---

## Grace Period

"Give me 15 min" calls `ActivityService.applyGracePeriod(instanceId)` which sets `graceUntil = now + 15 minutes`.

- One grace per instance — exits silently if already set
- Prevents snooze-loop behavior
- After grace expires, `CommitmentSchedulerService` detects expiry and publishes `GraceExpiredEvent`. `NotificationService` sends follow-up with two options:

```
Did you do it?
[ Yes, mark as done ]   [ No, skip this time ]
```

---

## Instance Fields Used

```
graceUntil: DateTime?          // set once by ActivityService.applyGracePeriod()
notifiedWindowClose: bool      // prevents duplicate check-in notifications
notifiedGrace: bool            // prevents duplicate grace follow-up notifications
```

---

## Rules

- Never fires for avoid commitments
- Never fires for frozen commitments
- One check-in per instance — tracked via `notifiedWindowClose`
- One grace per instance — `graceUntil` never updated after set
- Grace follow-up fires exactly once — tracked via `notifiedGrace`

---

## Dependencies

- `CommitmentSchedulerService` — publishes `WindowClosedWithNoLogEvent` and `GraceExpiredEvent`
- `NotificationService` — subscribes to events, sends notifications
- `ActivityService` — handles user responses (kept, grace, missed)