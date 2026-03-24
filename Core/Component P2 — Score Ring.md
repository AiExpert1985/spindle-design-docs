**File Name**: component_score_ring **Feature**: Core **Phase**: 2 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** displays any ratio as a circular arc with a sweep animation. Fully generic — receives a value and an optional label, draws the arc, knows nothing else. Can be placed anywhere a visual representation of a ratio is useful.

---

## Interface

```
ScoreRing(
  value: double,       // required — 0.0 to 1.0
  label: String?,      // optional — shown at center of the ring
  size: large | small, // optional — default large
)
```

The caller is responsible for fetching the value and deciding what label to show. The ring renders what it receives — no calculations, no data fetching.

---

## Visual

```
        ╭────────╮
       ╱  ██████  ╲
      │    83%     │
      │  this week │
       ╲            ╱
        ╰────────╯
```

- Clockwise arc from 12 o'clock
- Arc fills proportionally to `value`
- Percentage derived from `value` shown in large bold font at center
- Optional label in small secondary text below the percentage
- Arc color by value:

|Value|Color|
|---|---|
|< 0.50|Red|
|0.50–0.79|Amber|
|0.80–0.89|Green|
|≥ 0.90|Gold|

---

## Animation

**On first render:** sweeps from 0 to value over ~1 second.

**On value update:** animates from current value to new value over 400ms. Never jumps. Never re-sweeps from zero mid-session.

If animation system is under load — renders static at current value. See `ux_principles` rule 5.

---

## Size Variants

**Large** — primary placement, prominent visual on a screen. Minimum 120px diameter.

**Small** — secondary placement, inside a card or compact view.

---

## Expected Placements

|Placement|Value|Label|
|---|---|---|
|Dashboard — overall today|Today's overall day score|"today"|
|Dashboard — overall this week|This week's overall score|"this week"|
|Commitment Detail — today's commitment score|This commitment's `livePerformance` for today|Commitment name or "today"|
|Commitment Detail — this week's commitment score|This commitment's week score|"this week"|
|Day Celebration screen|Today's overall day score at celebration time|"today"|

Adding new placements requires zero changes to this component.

---

## Rules

- Pure UI — no service calls, no data fetching, no event subscriptions
- Receives data from parent — never fetches independently
- `value` clamped to 0.0–1.0 at render time — caller errors do not crash the ring

---

## Later Improvements

**Celebration mode.** Slightly slower sweep animation (~1.5 seconds) for use on the Day Celebration screen. Controlled by an optional `celebrationMode: bool` parameter.

**Possible value arc.** A secondary lighter arc showing the maximum achievable value — e.g. "you could still reach 92%". Requires a `possibleValue` parameter and a `getPossibleScore()` function from the caller's data source.