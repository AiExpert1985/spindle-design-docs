
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 1
**Standalone:** yes — can be added or removed without touching other components.

**Purpose:** holds soft-deleted commitments. Users can restore them or permanently delete them with all their history.

**Access:** Settings → Recycle Bin

---

## UI

```
┌─────────────────────────────────────────┐
│  RECYCLE BIN                            │
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

Empty state: `"No deleted commitments."`

---

## Actions

**Restore** Returns commitment to active state. All history, instances, and log entries preserved exactly as they were.

**Delete permanently** Removes the commitment definition, all its instances, and all log entries. Irreversible. Requires confirmation dialog:

```
Delete "Morning Walk" permanently?
This removes all history and cannot be undone.

[ Delete ]   [ Cancel ]
```

---

## Service Logic

Calls `CommitmentService` only — never touches the repository directly.

- Load list → `CommitmentService.getDeletedCommitments()`
- Restore → `CommitmentService.restoreCommitment(id)`
- Permanent delete → `CommitmentService.permanentDeleteCommitment(id)`

---

## Dependencies

- `CommitmentService` — all operations go through the commitment service