**File Name**: component_log_reward_animation **Feature**: Encouragement **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** instant visual feedback on every positive log. Confirms the action was recorded and makes logging feel satisfying. Purely ephemeral — nothing stored, nothing persisted.

Expected placement: overlaid at the tap point on any logging surface. Currently anticipated on the Dashboard card and the log entry dialog.

---

## Animation

A value label floats up from the tap point and fades out.

```
        +5 pages
           ↑
    [ Morning Walk        daily ]
```

- Rises ~80px from the tap point
- Fades over ~600ms
- Never blocks user input — tap-through at all times
- If animation system is under load — drop entirely. See `ux_principles` rule 5.

---

## Label Rules

The label always shows what changed — never a naked number, never "Logged."

|Commitment|Label|
|---|---|
|Binary (target: 1, no unit)|`✓`|
|Measurable (has unit)|`+{value} {unit}` — e.g. `+5 pages`, `+0.5km`|

Binary commitments always show `✓`, never `+1`. This avoids confusion with progression point language which also uses numeric values.

---

## When It Fires

Fires on any log that moves a commitment in the right direction.

**Does not fire on:**

- Miss or skip
- "I slipped" on a zero-tolerance avoid commitment
- Avoid limit exceeded
- Grace period skip

---

## Rules

- Purely presentational — no service calls, no storage, no events
- Receives logged value and commitment type from the parent surface — fetches nothing independently
- Ephemeral — plays once and is gone

---

## Later Improvements

**Haptic feedback.** `HapticFeedback.lightImpact()` fires instantly before the animation starts. Defined in `ux_principles` rule 1 as the first step of the logging moment sequence. Trivial to add — deferred until the full logging moment is designed on the Dashboard.