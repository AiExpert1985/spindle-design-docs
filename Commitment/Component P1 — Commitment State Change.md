**File Name**: component_commitment_state_change **Feature**: Commitment **Phase**: 1 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** owns the full state transition flow for a commitment — the ⋮ menu, the confirmation dialog, and the service call. The detail screen places this component and passes the commitment; the component handles everything from there.

Expected placement: top-right of the Commitment Detail screen, rendered as the ⋮ menu trigger.

Restore (deleted → active) is handled by the Recycle Bin screen — see `screen_recycle_bin`. This component covers the lifecycle transitions available while the commitment is visible in normal views.

---

## ⋮ Menu — Valid Transitions by State

The menu shows only the transitions valid for the commitment's current state. Invalid transitions are never shown.

**Active:**

```
Edit
Freeze
Mark as Complete
Delete
```

**Frozen:**

```
Edit
Unfreeze
Delete
```

**Completed:**

```
Edit
Delete
```

---

## Confirmation Dialogs

Each transition shows a confirmation dialog before calling the service.

**Freeze:**

```
Freeze "Morning Walk"?

This commitment will be paused.
History is preserved. Resume any time.

[ Freeze ]   [ Cancel ]
```

**Unfreeze:**

```
Resume "Morning Walk"?

This commitment will become active again
from today.

[ Resume ]   [ Cancel ]
```

**Mark as Complete:**

```
Mark "Morning Walk" as complete?

You're done with this commitment.
Your history is preserved permanently.

[ Mark Complete ]   [ Cancel ]
```

**Delete:**

```
Delete "Morning Walk"?

It will be moved to the recycle bin.
You can restore it any time.

[ Delete ]   [ Cancel ]
```

On cancel — dialog closes, nothing changes.

---

## Service Calls

|Transition|Service call|
|---|---|
|Active → Frozen|`CommitmentService.freezeCommitment(id)`|
|Frozen → Active|`CommitmentService.unfreezeCommitment(id)`|
|Active → Completed|`CommitmentService.completeCommitment(id)`|
|Active → Deleted|`CommitmentService.deleteCommitment(id)`|
|Frozen → Deleted|`CommitmentService.deleteCommitment(id)`|
|Completed → Deleted|`CommitmentService.deleteCommitment(id)`|

Edit navigates to the Commitment Form — not a state transition, but included in the menu as it belongs to the same management surface.

---

## Rules

- Only valid transitions for the current state are shown — never show an impossible action
- Delete is available from any non-deleted state — a user must always be able to remove a commitment
- Completed is a terminal forward state — no forward transitions beyond delete
- No state is changed until the user confirms the dialog

---

## Dependencies

- `CommitmentService` — all state transition calls
- `CommitmentService.getDefinition(id)` — reads current state to determine valid menu items

---

## Later Improvements

**Completion celebration.** A brief animation on confirm when marking a commitment as complete — acknowledging the achievement. Phase 3.

**Freeze with end date.** An optional date picker in the freeze dialog — "resume automatically on this date." Requires the Scheduled Freeze component. See `component_scheduled_freeze`.