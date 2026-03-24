**File Name**: component_garment_display **Feature**: Garment **Phase**: 3 **Created**: 18-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** renders the garment visual for one commitment. Receives a `GarmentProfile` and renders it using `GarmentRenderer`. Knows nothing about how the completion percent was calculated or what commitment it belongs to. The visual representation of long-term weaving progress.

Expected placement: Commitment Detail screen, replacing the plain performance number in Phase 3. Listed in that screen's Later Improvements section.

---

## Interface

```
GarmentDisplay(
  profile: GarmentProfile,     // required — type, colors, completionPercent
  weeklyDelta: double?,        // optional — highlights threads added this week
  size: large | small,         // optional — default large
  animateOnLoad: bool,         // optional — default true
)
```

All data passed in by the parent screen — the component fetches nothing independently.

---

## The Two Directions

**Do commitment** — garment begins empty, fills toward complete.

- 0% = bare loom, no threads
- 50% = half-woven, clearly in progress
- 100% = complete garment, every thread in place

**Avoid commitment** — garment begins complete, empties toward bare.

- 100% = complete garment — the habit has full presence
- 50% = half-unraveled — the grip is loosening
- 0% = bare loom — the grip is minimal

`commitmentType` from the profile determines which direction the fill reads. The renderer receives this and handles the direction.

---

## Rendering

Delegates entirely to `GarmentRenderer`. The component passes the profile to the renderer and displays the result — it has no knowledge of how the drawing works.

```
GarmentRenderer (abstract)
  → SimpleThreadRenderer        // flat thread fill, visible colored strands
  → (future: detailed woven texture, highlighted new threads, completion glow)
```

Thread colors come from `GarmentProfile.threadColors` — the same colors always appear for the same commitment, making each garment feel personal and consistent.

---

## Animation

**On first render:** threads animate in from 0% (or 100% for Avoid) to `completionPercent` over ~1.5 seconds. Plays every time the screen opens.

**On value update:** new threads animate in smoothly over ~600ms. Additive — threads visibly join the weave.

**On decay (percent decreases):** threads fade and withdraw. Subtle and slow — consistent with the UX principle of silence on negative events. ~800ms, plays on next screen open, not immediately.

**On load failure or animation under load:** renders static garment at current percent. Never blocks, never shows an error. See `ux_principles` rule 5.

---

## Weekly Delta Highlight (when provided)

When `weeklyDelta > 0`, the threads added this week are rendered in a slightly brighter shade — visually distinguishable from older threads but not jarring. A subtle label appears below: "This week's work."

For Avoid commitments: the threads removed this week show a gentle fade on the unraveled section.

Requires no model changes — `weeklyDelta` comes from `GarmentService.getLiveWeekDelta()`.

---

## Completion State

When `completionPercent == 100.0` (Do) or `completionPercent == 0.0` (Avoid): garment renders in a distinct completion state — warmer color, subtle glow. Plays only the first time completion is reached, not on repeat views.

The completion label is rendered by the host screen, not this component — keeps the component generic.

---

## Size Variants

**Large** — primary placement on commitment detail screen. Minimum 200px tall.

**Small** — reserved for future summary cards or widgets.

---

## Rules

- Pure visual component — no service calls, no repository access, no event subscriptions
- Receives all data from parent — never fetches independently
- Delegates all drawing to `GarmentRenderer` — contains no canvas logic
- `GarmentRenderer` is injected — the component never instantiates it directly

---

## Dependencies

- `GarmentRenderer` — injected, handles all canvas drawing
- Parent screen — provides `GarmentProfile` and `weeklyDelta`
