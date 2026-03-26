**File Name**: screen_recycle_bin **Feature**: UserSettings **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** shows soft-deleted commitments. The user can restore them or permanently delete them with all their history. A housekeeping screen — visited rarely and deliberately.

Accessed via Settings → Recycle Bin. Not in bottom nav.

Placed in Settings rather than the Commitments screen because it is a housekeeping action — visited rarely and deliberately, never as part of the daily flow. Keeping it in Settings preserves the Commitments screen as a clean daily management surface and matches the user's expectation that destructive or recovery actions live in a dedicated area, not inline with the main list.

---

## Layout

```
┌─────────────────────────────────────────┐
│  ←  Recycle Bin                         │
│                                         │
│  Morning Walk                           │
│  Deleted 3 days ago                     │
│  [ Restore ]  [ Delete permanently ]    │
│                                         │
│  No alcohol                             │
│  Deleted 12 days ago                    │
│  [ Restore ]  [ Delete permanently ]    │
└─────────────────────────────────────────┘
```

Empty state: "No deleted commitments."

---

## Actions

**Restore** — returns the commitment to active state. All history, instances, and log entries preserved exactly as they were.

**Delete permanently** — removes the commitment definition, all its instances, and all log entries. Irreversible. Requires confirmation:

```
Delete "Morning Walk" permanently?
This removes all history and cannot be undone.

[ Delete ]   [ Cancel ]
```

---

## Navigation

- Accessed from: Settings screen → Recycle Bin
- Back → returns to Settings screen

---

## Data Sources

|Data|Source|
|---|---|
|Deleted commitments|`CommitmentService.watchDeletedCommitments()` — stream|
|Restore|`CommitmentService.restoreCommitment(id)`|
|Permanent delete|`CommitmentService.permanentDeleteCommitment(id)`|