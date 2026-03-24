**File Name**: loghistorysheet **Feature**: Activity **Phase**: 1 **Created**: 21-Mar-2026 **Modified**: -

---

**Purpose:** shows the log history for one commitment on a given date. Allows the user to edit or delete entries within the backfill window. Opened from the commitment detail screen.

---

**Expected placement:** opened as a bottom sheet from the screen that shows the full detail of one commitment, via a "History" button or link near the log entry area.

---

## Parameters

```
LogHistorySheet(
  definitionId: String,
  commitmentName: String,
  commitmentType: CommitmentType,
  target: Target,
  date: Date
)
```

Opens showing all log entries for `definitionId` on `date`, ordered by `loggedAt` descending. The sheet is read-only for entries older than `AppConfig.maxLogBackfillDays` — edit and delete buttons are not shown for those entries.

---

## Entry Display

Each entry shows:

- Value with unit label (e.g. "3.2 km", "50 pages", "2 cups")
- Time of `loggedAt`
- Note if present
- Context tags if present

For entries within the backfill window, two actions are shown:

- **Edit** — opens `LogEntryDialog` pre-filled with the entry's current `value` and `note`. `dateLocked: true`, `date` fixed to the entry's `loggedAt` date. Only value and note are editable.
- **Delete** — removes the entry after a single confirmation tap. No undo.

For entries outside the backfill window, no actions are shown — display only.

---

## Empty State

If no entries exist for the given date, shows a brief message: "No activity logged for this day."

---

## Rules

- Calls `ActivityService.editEntry()` and `ActivityService.deleteEntry()` — no direct repository access
- Edit opens `LogEntryDialog` — no separate edit screen
- Delete requires one confirmation tap — no undo
- Read-only for entries outside the backfill window — no edit or delete shown
- The sheet itself does not validate the backfill window — ActivityService enforces this on every write
- Opened from commitment detail screen only — not reachable from anywhere else

---

## Dependencies

- ActivityService — reads entries, calls editEntry and deleteEntry
- LogEntryDialog — reused for editing
- AppConfig — maxLogBackfillDays (for showing/hiding edit and delete buttons)