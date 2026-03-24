**File Name**: component_commitment_state_change **Feature**: Commitment **Phase**: 1 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** owns the full state transition flow for a commitment — the ⋮ menu, the confirmation dialog, and the service call. The detail screen places this component and passes the commitment; the component handles everything from there.

Expected placement: top-right of the Commitment Detail screen, rendered as the ⋮ menu trigger.

Soft delete is not handled here — it is a destructive action with different intent and lives as a separate action elsewhere. This component covers lifecycle transitions only.

---

## ⋮ Menu — Valid Transitions by State

The menu shows only the transitions valid for the commitment's current state. Invalid transitions are never shown.

**Active:**

```
Edit
Freeze
Mark as Complete
```

**Frozen:**

```
Edit
Unfreeze
```

**Completed:**

```
(no state transitions available)
Edit
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

On cancel — dialog closes, nothing changes.

---

## Service Calls

|Transition|Service call|
|---|---|
|Active → Frozen|`CommitmentService.freezeCommitment(id)`|
|Frozen → Active|`CommitmentService.unfreezeCommitment(id)`|
|Active → Completed|`CommitmentService.completeCommitment(id)`|

Edit navigates to the Commitment Form — not a state transition, but included in the menu as it belongs to the same management surface.

---

## Rules

- Only valid transitions for the current state are shown — never show an impossible action
- Soft delete is excluded — handled separately
- Completed is a terminal state — no forward transitions, menu shows Edit only
- No state is changed until the user confirms the dialog

---

## Dependencies

- `CommitmentService` — all state transition calls
- `CommitmentService.getDefinition(id)` — reads current state to determine valid menu items

---

## Later Improvements

**Completion celebration.** A brief animation on confirm when marking a commitment as complete — acknowledging the achievement. Phase 3.

**Freeze with end date.** An optional date picker in the freeze dialog — "resume automatically on this date." Requires the Scheduled Freeze component. See `component_scheduled_freeze`.