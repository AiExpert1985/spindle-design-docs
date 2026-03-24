
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 1

**Standalone:** yes — used on Dashboard and Threads screen.

**Purpose:** displays one commitment's current state and is the primary surface for logging. The most interacted-with element in the app.

---

## Layout

```
[ Morning Walk    daily    ██████░░  60%  ⚠ ]
```

- Left: commitment name + recurrence label
- Center: progress bar fills proportionally to `progressPercent`
- Right: percentage + warning indicator if window closing
- Full row is swipeable and tappable

---

## States

**Active — in progress:**

```
[ Morning Walk    daily    ██████░░  60% ]
```

**Active — kept (100%):**

```
[ Morning Walk    daily    ████████ 100% ✓ ]
```

Progress bar full, green, checkmark.

**Active — warning (window closing):**

```
[ Morning Walk    daily    ██████░░  60% ⚠ ]
```

Warning icon appears when 3/4 of window has elapsed.

**Zero-tolerance avoid:**

```
[ No alcohol      daily    ████████ 100% ✓ ]
[ I slipped ]
```

Binary state. "I slipped" button always visible.

**Frozen:**

```
[ Morning Walk    ❄ frozen              ]
```

Grey, no progress bar, no swipe gestures.

**Completed:**

```
[ Morning Walk    ✓ completed           ]
```

Muted, read-only.

---

## Gestures (Active cards only)

- **Swipe right** → quick-log + haptic + floating value animation
- **Swipe left** → failure recovery sheet
- **Long press** → full log sheet
- **Tap** → navigates to Thread Detail screen
- Frozen and completed cards: tap only — no swipe gestures

---

## Progress Bar

- Fills smoothly on every log — 400ms, never jumps
- Color matches score ring threshold (red / amber / green / gold)
- For avoid commitments: bar fills as usage approaches limit — full = at limit

---

## Dashboard vs Threads Screen Variant

**Dashboard variant:** shows gestures, progress bar, logging interaction. Full interactivity.

**Threads screen variant:** shows percentage and trend indicator only. No swipe gestures — viewing only.

---

## Dependencies

- `CommitmentService.getTodayInstances()` — instance data for progress
- Commitment logging component — handles swipe gesture actions
- Context tags component — tag sheet shown after certain log events