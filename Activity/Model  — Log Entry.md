**File Name**: logentry **Feature**: Activity **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** one record of user activity against a commitment. Entries may be corrected within the backfill window but never deleted individually.

---

## Design: definitionId + date, No instanceId

LogEntry carries `definitionId` and `loggedAt` — not `instanceId`. PerformanceService identifies the correct instance by finding the instance whose `regenerationWindow` contains `loggedAt` for the given commitment.

This decouples the log from instance lifecycle — if an instance is cleared and recreated, existing logs are still correctly attributed by date. Grace-period logs work naturally: a log recorded after midnight finds the previous day's closed instance by date, with no special handling.

---

## Date Restriction

Logs may only be recorded or edited for today and up to `AppConfig.maxLogBackfillDays` (default: 7) days in the past. Future logs are never accepted. Entries older than the backfill window become read-only — they can be viewed in the log history but not changed.

---

## Fields

```
LogEntry
  id: String
  definitionId: String
  value: double
  loggedAt: DateTime
  note: String?
  contextTags: List<String>
  createdAt: DateTime
  updatedAt: DateTime?
```

- **id** — client-generated UUID. Immutable.
- **definitionId** — which commitment this log belongs to.
- **value** — the amount recorded. Must be > 0. Examples: 3.2 (km run), 50 (pages read), 1 (binary done), 2 (cups for an avoid commitment).
- **loggedAt** — the date the user attributes this activity to. May differ from `createdAt` for grace-period logs or backfill entries.
- **note** — optional free-text. Used by AI Insights in later phases.
- **contextTags** — optional predefined tags (tired, social, travel, motivated). Used for pattern analysis by AI Insights. Empty list if none provided.
- **createdAt** — when this record was first written. Immutable.
- **updatedAt** — when this record was last edited. Null if never edited. Distinct from `createdAt` — set only on edit, not on creation.

---

## Rules

- `value` must be > 0
- `loggedAt` must be within `AppConfig.maxLogBackfillDays` — never future
- Entries within the backfill window may be edited (value and note only) or deleted
- Entries older than the backfill window are read-only
- `loggedAt` is immutable after creation — moving a log to a different date is not allowed
- Bulk deletion only on permanent commitment deletion
- No `instanceId` — instance identified at read time by `definitionId + loggedAt` date
- Owned by ActivityRepository only