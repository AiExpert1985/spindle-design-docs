
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 1

**Standalone:** yes — pure UI component, no service dependencies.

**Purpose:** displays any percentage score as a circular arc with a sweep animation. Fully generic — receives a number and animates to it. Has no knowledge of where the number comes from. Can be placed on any screen.

**Feature:** Core

---

## Interface

The ring accepts a value and optional secondary value:

```
ScoreRing(
  value: 83,            // required — 0 to 100
  possibleValue: 92,    // optional — shows "→ up to 92%" below center
  size: large | small,  // optional — default large
)
```

Nothing else. The caller is responsible for fetching the number and passing it in.

---

## Visual

```
        ╭────────╮
       ╱  ██████  ╲
      │    83%     │
      │  →92% poss │
       ╲            ╱
        ╰────────╯
```

- Clockwise arc from 12 o'clock
- Score percentage in large bold font at center
- Optional possible score in small secondary text below
- Arc color by value:

|Value|Color|
|---|---|
|< 50%|Red|
|50–79%|Amber|
|80–89%|Green|
|≥ 90%|Gold|

---

## Animation Behavior

**On first render:** sweeps from 0% to value over ~1 second. Plays every time the ring appears — gives energy to any screen it's placed on.

**On value update:** animates from current value to new value over 400ms. Never jumps. Never re-sweeps from zero mid-session.

**On celebration context:** same sweep, slightly slower (~1.5 seconds) — caller controls this via an optional `celebrationMode: bool` parameter.

---

## Where It's Used

|Screen|Value passed|Who fetches it|
|---|---|---|
|Dashboard|Today's overall score|Dashboard screen|
|Commitment Detail|This commitment's score today|Commitment Detail screen|
|Day Celebration|Today's score at celebration time|Day Celebration component|

Adding new uses requires zero changes to this component.

---

## Size Variants

**Large** — primary placement, top of screen. Minimum 120px diameter.

**Small** — secondary placement, inside a card or compact view. Future use.

---

## Dependencies

None. Pure UI component.