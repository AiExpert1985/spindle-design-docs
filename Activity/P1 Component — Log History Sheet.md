**File Name**: component_log_history_sheet **Feature**: Activity **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** shows the log history for one commitment. Allows the user to edit or delete entries within the backfill window. Opened from the commitment detail screen.

---

**Expected placement:** opened as a bottom sheet from the screen that shows the full detail of one commitment, via the "View Log History" button.

---

## Parameters

```
LogHistorySheet(
  definitionId: String,
  commitmentName: String,
  commitmentType: CommitmentType,
  target: Target,
  initialDate: Date
)
```

Opens showing entries for `initialDate` (today by default). The user can switch between day view and week view using the toggle at the top.

---

## Views

### Day View

Shows all log entries for `definitionId` on the selected date, ordered by `loggedAt` descending. A date picker at the top allows navigation within the last `AppConfig.maxLogBackfillDays` days — no future dates selectable.

### Week View

Shows all log entries for `definitionId` across the Mon–Sun week containing the selected date, grouped by day and ordered by `loggedAt` descending within each group. Uses `ActivityService.getEntriesForWeek()`.

A toggle at the top switches between views. The selected date carries across — switching to week view shows the week containing the currently selected day.

---

## Entry Display

Each entry shows:

- Value with unit label (e.g. "3.2 km", "50 pages", "2 cups")
- Time of `loggedAt`
- Note if present

For entries within the backfill window, two actions are shown:

- **Edit** — opens `LogEntryDialog` pre-filled with the entry's current `value` and `note`. `dateLocked: true`. Only value and note are editable.
- **Delete** — removes the entry after a single confirmation tap. No undo.

For entries outside the backfill window, no actions are shown — display only. A subtle label reads "Read only — outside edit window" to explain the missing actions without alarming the user.

---

## Empty State

No entries for the selected day or week:

```
No activity logged for this period.
```

---

## Error Handling

If `editEntry` or `deleteEntry` returns a `Failure` due to the backfill window, the sheet shows:

```
This entry can no longer be edited.
Only entries from the last 7 days can be changed.
```

The sheet stays open. The entry reverts to display-only.

---

## Rules

- Calls `ActivityService.getEntriesForDay()`, `getEntriesForWeek()`, `editEntry()`, `deleteEntry()` — no direct repository access
- Edit opens `LogEntryDialog` — no separate edit screen
- Delete requires one confirmation tap — no undo
- The sheet does not enforce the backfill window itself — `ActivityService` enforces it on every write
- On `Failure` from `ActivityService` — show the user the reason, keep sheet open
- Date navigation limited to `AppConfig.maxLogBackfillDays` — no future dates

---

## Dependencies

- `ActivityService` — all reads and writes
- `LogEntryDialog` — reused for editing
- `AppConfig` — `maxLogBackfillDays`