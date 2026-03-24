
**Created**: 18-Mar-2026 **Modified**: - **Feature**: Commitment **Phase**: 2 **Standalone:** yes — pure visual component. Can be placed on any screen that needs to show a commitment's garment progress.

**Purpose:** renders the garment visual for one commitment. Receives a completion percentage and a garment type — knows nothing about how the percentage was calculated or what commitment it belongs to. The visual representation of long-term weaving progress.

---

## Interface

```
GarmentDisplay(
  completionPercent: double,      // required — 0.0 to 100.0
  garmentType: GarmentType,       // required — determines which garment is rendered
  commitmentType: CommitmentType, // required — do or avoid, determines direction
  weeklyDelta: double?,           // optional — used in Phase 3 to highlight new threads
  size: large | small,            // optional — default large
  animateOnLoad: bool,            // optional — default true
)
```

Nothing else. The caller fetches the data and passes it in. This component makes no service calls.

---

## The Two Directions

**Do commitment** — garment begins empty, fills toward complete.

- 0% = bare loom, no threads
- 50% = half-woven garment, clearly in progress
- 100% = complete garment, every thread in place

**Avoid commitment** — garment begins complete, empties toward bare.

- 100% = complete garment — the habit has full presence
- 50% = half-unraveled — the grip is loosening
- 0% = bare loom — the grip is minimal

The visual rendering is the same component. The `commitmentType` parameter determines which direction the fill reads.

---

## Garment Visuals

Each garment type has a distinct silhouette rendered as a custom Flutter canvas widget. Thread colors come from `CommitmentDefinition.threadColors` — the same colors used in the rope visual on the dashboard, maintaining identity across views.

|GarmentType|Visual|Empty state|Complete state|
|---|---|---|---|
|thread_bracelet|Circular wrist band|Bare cord|Fully braided|
|sock|Sock silhouette|Bare outline|Fully filled|
|glove|Glove silhouette|Bare outline|Fully filled|
|scarf|Long rectangular wrap|Bare outline|Rich layered texture|
|sweater|Sweater silhouette|Bare outline|Fully woven|

The fill is not a simple color overlay — it shows visible thread texture, making the weaving metaphor literal. Threads are visible as individual colored strands filling the silhouette.

---

## Animation Behavior

**On first render:** threads animate in from 0% to `completionPercent` over ~1.5 seconds. Plays every time the detail screen opens — gives the screen energy and reinforces the weaving metaphor.

**On value update (after a log):** new threads animate in smoothly from current percent to updated percent. The animation is additive — threads visibly join the weave. Duration: ~600ms.

**On decay (percent decreases):** threads fade and withdraw. No dramatic animation — subtle, slow, consistent with the UX principle of silence on negative events. Duration: ~800ms, plays on next screen open, not immediately.

**On load failure / animation system under load:** renders static garment at current percent. Never blocks or shows an error — the number below the garment remains accurate.

---

## Phase 3 Enhancement — Highlighted New Threads

When `weeklyDelta` is provided and greater than 0, the threads added this week are rendered in a slightly brighter shade — visually distinguishable from older threads but not jarring. A subtle label appears below: _"This week's work."_

This enhancement requires no data model changes — `weeklyDelta` is already calculated and stored by `GarmentCalculationService`. The display component simply uses it when available.

For avoid commitments: the threads removed this week are shown with a gentle fade effect on the unraveled section.

---

## Completion State

When `completionPercent == 100.0` (do habit) or `completionPercent == 0.0` (avoid habit):

The garment renders in a distinct completion state — slightly warmer color, subtle glow on the thread texture. No animation plays on repeat views — only on the first time the completion state is reached.

The completion label is rendered below the garment by the screen that hosts this component, not by the component itself. This keeps the component generic.

---

## Size Variants

**Large** — primary placement on commitment detail screen. Minimum 200px tall.

**Small** — future use. Reserved for summary cards or widgets that need a compact garment indicator.

---

## Where It Is Used

|Screen|Purpose|Data source|
|---|---|---|
|Commitment Detail|Primary visual — replaces Score Ring|`GarmentCalculationService` → definition|

Adding new uses requires zero changes to this component.

---

## Dependencies

None. Pure visual component. No service calls, no repository access, no business logic.