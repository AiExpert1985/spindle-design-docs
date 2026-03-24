
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 2
**Standalone:** yes — can be added or removed without touching other components.

**Purpose:** instant visual feedback on every positive log. Confirms the action was recorded and makes logging feel satisfying. Purely ephemeral — nothing stored.

---

## UI

A value label floats up from the tap point and fades out.

```
        +5 pages
           ↑
    [ Morning Walk  ████░░  60% ]
```

- Label shows the logged value in context: `+1`, `+0.5km`, `+5 pages`, `✓`
- Floats upward ~80px from the tap point
- Fades out over ~600ms
- Green color
- Never blocks user input — tap-through

---

## When It Fires

- Any log that moves a commitment in the right direction
- Swipe-right quick-log
- Manual log from the log sheet

**Does not fire on:**

- Miss or skip
- "I slipped" on avoid commitment
- Avoid limit exceeded
- Grace period skip

---

## Service Logic

None — purely presentational. The presentation layer triggers this directly after a successful log write, with no service involvement.

---

## Dependencies

- Presentation layer only — watches log state via Riverpod
- No repository access
- No scheduler