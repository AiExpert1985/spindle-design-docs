**File Name**: model_log_entry **Feature**: Activity **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** one record of user activity against a commitment. Entries may be corrected within the backfill window but are read-only beyond it.

---

## Why value is Always > 0

A log entry only exists when the user recorded something happening. For do commitments this means activity occurred. For avoid commitments this means the user did the thing they were trying to avoid — the logged value is how much they did.

Zero is never written. Absence of a log is the signal for zero activity on a given day. For avoid commitments, no log means the user succeeded — they avoided entirely. This keeps the model clean: a record's existence always means something was recorded.

---

## Backfill Window

Logs may only be recorded or edited for today and up to `AppConfig.maxLogBackfillDays` (default: 7) days in the past. Future logs are never accepted. Entries older than the backfill window become read-only.

---

## Fields

```
LogEntry
  id: String
  definitionId: String
  value: double
  loggedAt: DateTime
  note: String?
  createdAt: DateTime
  updatedAt: DateTime
```

- **id** — client-generated UUID. Immutable.
- **definitionId** — which commitment this log belongs to.
- **value** — the amount recorded. Always > 0. Examples: 3.2 (km run), 50 (pages read), 1 (binary done), 2 (cups for an avoid commitment).
- **loggedAt** — the date the user attributes this activity to. May differ from `createdAt` for backfill entries. Immutable after creation.
- **note** — optional free-text. Hidden in Phase 1 UI — stored for future AI Insights use.
- **createdAt** — when this record was first written. Immutable.
- **updatedAt** — set to `createdAt` on creation. Updated on every edit. Never null.

---

## Rules

- `value` must be > 0 — a log entry always represents something that happened
- `loggedAt` must be today or within `AppConfig.maxLogBackfillDays` — never future
- `loggedAt` is immutable after creation — moving a log to a different date is not allowed
- `updatedAt` is always set — initialized to `createdAt` on creation, updated on edit
- Entries within the backfill window may be edited (value and note only) or deleted
- Entries older than the backfill window are read-only

---

## Later Improvements

**Context tags.** A list of predefined tags (tired, social, travel, motivated) the user can attach to a log entry at the moment of recording. Transforms raw activity data into behavioral data — enabling Analytics and AI Insights to explain patterns rather than just describe them. Requires adding `contextTags: List<String>` to this model and a tag selection UI in the log entry dialog. See `component_context_tags`.