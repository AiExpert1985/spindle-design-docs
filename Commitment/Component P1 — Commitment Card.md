**File Name**: component_commitment_card **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** displays one commitment as a row in the commitments list. Shows name, recurrence, and current status at a glance. Read-only — no logging interaction.

Expected placement: Commitments screen, inside the Do and Avoid groups.

---

## Layout

```
[ Morning Walk       daily                       ]
[ Read 30 pages      weekly                      ]
[ No alcohol         daily                       ]
[ Morning Walk       ❄ frozen                    ]
[ Morning Walk       ✓ completed                 ]
```

- Left: commitment name
- Right: recurrence label — or state label for frozen / completed

Frozen and completed rows show their state label in place of the recurrence label.

---

## Gestures

Active rows:

- **Tap** → Commitment Detail screen
- **Swipe left** → soft delete — silent, no confirmation. Undo snackbar appears briefly. Commitment moves to Recycle Bin.
- **Long press** → action menu: Edit · Freeze · Delete

Frozen rows: tap only.

Completed rows: tap only.

---

## Dependencies

- `CommitmentService.watchActiveCommitments()` / `watchFrozenCommitments()` / `watchCompletedCommitments()` — commitment list data, provided by the parent screen

The card receives its data from the parent screen's provider — it does not fetch independently.

---

## Later Improvements

**Status icon.** Shows current window state at a glance — ✓ complete, ⚠ window closing, ✕ not a scheduled day. Requires `CommitmentIdentityService.watchInstancesForDay(today)` added to the parent screen's provider.

**Progress bar.** A visual fill showing `livePerformance` for the current window. Requires the Performance feature. See Commitments screen later improvements.

**Performance trend indicator.** A small arrow or color shift indicating whether this commitment is improving, steady, or declining over recent windows. Requires the Performance feature.