
**Created**: 15-Mar-2026 
**Modified**: 17-Mar-2026 
**Feature**: Commitment 
**Phase**: 1 
**Standalone:** yes — core to the app, cannot be removed.

**Purpose:** how users record progress against their commitments. The highest-frequency interaction in the app — must be fast, rewarding, and never require navigation. Maximum 5 seconds from any screen to logged.

---

## Why Logging Is Designed This Way

Every extra tap or navigation step costs engagement. Research shows that habit apps fail when logging feels like work. The design principle here is: the fastest path to logged is the default path. Everything else is one step away.

---

## Logging Methods

### Swipe Right — Quick Log

The fastest path. Swipe right on any commitment card on the dashboard.

- Logs `defaultLogIncrement` immediately
- Haptic feedback fires instantly
- Floating label animation rises from tap point — see Feedback Sequence below for label rules
- No confirmation, no sheet — just done

### Swipe Left — Failure Recovery

Not a hard miss — a recovery moment. Opens a sheet immediately.

```
You skipped "Morning Walk"

[ busy ] [ forgot ] [ low energy ] [ other ]   ← context tags (Phase 2)

That's okay. Small step?
[ Log 10 min walk now ]        ← partial progress, logged immediately
[ Skip — I'll do it next time ] ← conscious skip, stays 0%
```

Partial progress is honest and counts — 10 min of a 30 min walk = 33%. For avoid threads already exceeded: show tags + "Got it" only — no recovery option.

### Long Press — Full Log Sheet

Opens a sheet with custom value input. For users who want precision.

```
Morning Walk
[ 3.2 ] km        ← editable value
Note: (optional)
[ Log ]
```

### "+" Button — Detail Screen Log

Same full log sheet, accessed from thread detail screen.

---

## Zero-Tolerance Avoid ("I Slipped")

When `commitmentType: avoid` and `target: 0`, the card shows a binary state:

```
No alcohol today     100% ✓
[ I slipped ]
```

Tapping "I slipped":

- Logs `value: 1`
- Sets instance to 0%
- Shows context tag sheet (Phase 2)
- No quantity prompt — binary, no recovery

---

## Feedback Sequence

After every successful log, in this exact order:

1. **Haptic** — instant, before any animation
2. **Floating label** — rises ~80px from the tap point, fades over 600ms
3. **Progress bar** — fills smoothly over 400ms, never jumps
4. **Score ring** — animates clockwise over 400ms
5. **Threshold message** — `"Halfway there."` / `"Good start."` fades after 2 seconds

**Floating label rules — never a naked number:**

|Commitment type|Label shown|
|---|---|
|Binary (target: 1, no unit)|`✓`|
|Measurable (has unit)|`+{value} {unit}` e.g. `+0.5km`, `+5 pages`, `+1 rep`|

Binary commitments always show `✓`, never `+1`. This prevents visual confusion with progression point language, which also uses numeric values in notifications and on the Progression Screen.

Never show `"Logged successfully."` — always show what changed:

```
15 → 20 pages · 64% → 70%
```

---

## Service Logic

- Swipe right → `CommitmentService.logEntry(instanceId, defaultLogIncrement)`
- Full log → `CommitmentService.logEntry(instanceId, value, note?)`
- Failure recovery partial → `CommitmentService.logEntry(instanceId, value, tags: tags)`
- "I slipped" → `CommitmentService.logEntry(instanceId, 1)`
- Conscious skip → `CommitmentService.markInstanceMissed(instanceId)`

After every log → `ScoringService` recalculates `progressPercent` → Riverpod state updates → UI animates.

---

## Dependencies

- `CommitmentService` — all log writes
- `ScoringService` — recalculates progress after each log
- Presentation layer — watches instance state via Riverpod, triggers feedback animations
- Context tags component (Phase 2) — tag sheet shown at specific moments