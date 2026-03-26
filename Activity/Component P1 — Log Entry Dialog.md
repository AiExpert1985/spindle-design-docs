**File Name**: logentrydialog **Feature**: Activity **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** the single entry point for recording user activity against a commitment. A lightweight dialog opened from two paths — manual navigation and notification action. Both paths lead to the same component with the same behaviour.

---

**Expected placement:** opened as a centered dialog from the screen that shows the full detail of one commitment. Also triggered directly by notification actions — in that case it is opened by the notification handler, not by navigation.

---

## Parameters

```
LogEntryDialog(
  definitionId: String,
  commitmentName: String,
  commitmentType: CommitmentType,
  target: Target,
  date: Date,
  dateLocked: bool
)
```

- **definitionId** — passed to ActivityService when writing the log entry.
- **commitmentName** — displayed so the user knows which commitment they are logging for.
- **commitmentType** — determines whether to show binary or numerical input.
- **target** — provides the target value and unit label. Example: "target 5 km", "limit 2 cups".
- **date** — the date this log is attributed to. Defaults to today when opened manually. Set to the window date when opened from a notification.
- **dateLocked** — when true, the date field is shown but not editable. Used for notification-driven logging.

The dialog is fully passive — it receives everything it needs and calls `ActivityService.recordEntry()` on confirmation. It contains no business logic and fetches nothing.

---

## Two Variants

### Binary (target.value == 1, no unit)

```
[ Morning Run ]
  Done ✓        Dismiss
```

Two actions. No value input. "Done ✓" calls `recordEntry(value: 1, loggedAt: date)`. Dismiss closes without writing — absence of a log is the miss signal.

### Numerical

```
[ Morning Run ]   target: 5 km
  [ 3.2    ] km
  ▸ Add note / tags   (collapsed — tap to expand)
  [ Log ]
```

Number input with unit label from `target.measureUnit`. Confirm button enabled only when value > 0. For avoid commitments the label reads "How much did you have?" instead of the default prompt — same input structure.

---

## Notes and Context Tags

Both are hidden by default behind a single "Add note / tags" toggle. Tapping it expands:

- **Note** — free-text field, optional. Used by AI Insights in later phases.
- **Context tags** — predefined list (tired, social, travel, motivated, etc.). Multi-select. Used for pattern analysis by AI Insights.

Hiding them by default keeps the dialog fast for the common case — most logs are quick value entries. Users who want to add context tap to expand. The expand state is not persisted between opens.

---

## Date Selection

When `dateLocked: false`, the user can change the date within the last `AppConfig.maxLogBackfillDays` days. Future dates are not selectable. This covers backfill scenarios — travel, illness, forgetting to log.

When `dateLocked: true` (notification path), the date field is visible but not interactive.

Logging is allowed regardless of the commitment's current state — frozen, completed, or active. A user may navigate to a frozen commitment and add backfill logs. The date restriction is the only gate, enforced by ActivityService.

---

## Opening Paths

**Manual — from commitment detail screen** Passes `definitionId`, `commitmentName`, `commitmentType`, `target`, `date: today`, `dateLocked: false`.

**Notification-driven — from window-close notification** Passes `definitionId`, `commitmentName`, `commitmentType`, `target`, `date: windowDate`, `dateLocked: true`.

Both paths open the same dialog. The only difference is `dateLocked`.

---

## Notification Actions

The window-close notification is constructed and scheduled by `CommitmentNotificationSchedulerService` — see that doc for the full payload definition. The dialog is only involved when the user taps "Log" on a numerical commitment.

**Binary commitment action — "Done":** calls `ActivityService.recordEntry(value: 1, loggedAt: windowDate)` directly. The dialog is not opened.

**Numerical commitment action — "Log":** opens this dialog with `dateLocked: true` and `date: windowDate`.

**"15 more minutes" action:** carries `actionId: 'grant_grace'` in the notification payload. `GraceService` subscribes to `NotificationActionEvent` and handles it. The dialog is not involved — this is a notification-level action, not a UI action.

**Dismiss** — no log written. No grace created.

---

## Rules

- The dialog calls only `ActivityService.recordEntry()` — nothing else
- All parameters passed by caller — dialog fetches nothing
- `dateLocked: true` when opened from notification
- Binary variant never writes a missed log — dismiss is always silent
- Dismiss always closes without writing
- Note and context tags hidden by default — expanded on user tap
- Logging allowed regardless of commitment state — frozen commitments can receive backfill logs