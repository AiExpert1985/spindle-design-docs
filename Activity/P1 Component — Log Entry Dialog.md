**File Name**: component_log_entry_dialog **Feature**: Activity **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the single entry point for recording user activity against a commitment. A lightweight dialog opened from two paths — manual navigation and notification tap. Both paths lead to the same component with the same behaviour.

---

**Expected placement:** opened as a centered dialog from the screen that shows the full detail of one commitment. Also opened when the user taps a window-close or checkin notification — in that case the app navigates to the activity log screen which opens the dialog directly.

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

- **definitionId** — passed to `ActivityService` when writing the log entry.
- **commitmentName** — displayed so the user knows which commitment they are logging for.
- **commitmentType** — determines whether to show binary or numerical input.
- **target** — provides the target value and unit label.
- **date** — the date this log is attributed to. Defaults to today when opened manually. Set to the window date when opened from a notification deep-link.
- **dateLocked** — when true, the date field is shown but not editable. Used for notification-driven logging.

The dialog is fully passive — it receives everything it needs and calls `ActivityService.recordEntry()` on confirmation. It contains no business logic and fetches nothing.

---

## Two Variants

### Binary (target.value == 1, no unit)

```
[ Morning Run ]
  Done ✓        Dismiss
```

Two actions. No value input. "Done ✓" calls `recordEntry(value: 1, loggedAt: date)`. Dismiss closes without writing.

### Numerical

```
[ Morning Run ]   target: 5 km
  [ 3.2    ] km
  ▸ Add note   (collapsed — tap to expand)
  [ Log ]
```

Number input with unit label from `target.measureUnit`. Confirm button enabled only when value > 0. For avoid commitments the label reads "How much did you have?" instead of the default prompt.

---

## Note

Hidden by default behind an "Add note" toggle. Tapping it expands a free-text field. Optional. The expand state is not persisted between opens.

Note is stored on the log entry for future AI Insights use — it has no effect on scoring or performance in Phase 1.

---

## Date Selection

When `dateLocked: false`, the user can change the date within the last `AppConfig.maxLogBackfillDays` days. Future dates are not selectable. The date picker shows only selectable dates — no interaction needed for the restriction in the common case.

When `dateLocked: true` (notification path), the date field is visible but not interactive.

If `ActivityService.recordEntry()` returns a `Failure` due to the date being outside the backfill window, the dialog shows:

```
This date is too far back to log.
You can only log activity for the last 7 days.
```

The dialog stays open — the user can change the date or dismiss.

---

## Opening Paths

**Manual — from commitment detail screen** Passes `definitionId`, `commitmentName`, `commitmentType`, `target`, `date: today`, `dateLocked: false`.

**Notification-driven — from window-close or checkin notification tap** The notification carries a deep-link route: `/log?definitionId=abc123`. The app navigates to the activity log screen, which opens this dialog with `dateLocked: true` and `date: windowDate`.

Both paths open the same dialog. The only difference is `dateLocked`.

---

## Rules

- The dialog calls only `ActivityService.recordEntry()` — nothing else
- All parameters passed by caller — dialog fetches nothing
- `dateLocked: true` when opened from notification
- Binary variant never writes a missed log — dismiss is always silent
- Dismiss always closes without writing
- Logging allowed regardless of commitment state — frozen commitments can receive backfill logs
- On `Failure` from `ActivityService` — show the user the reason, keep dialog open

---

## Later Improvements

**Context tags.** Predefined tags (tired, social, travel, motivated) attached to a log at the moment of recording. Shown as an optional step after confirmation. Requires `contextTags` field on `LogEntry` and `ActivityService.editEntry()` extension. See `component_context_tags`.