
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 1

**Standalone:** yes — can be added or removed without touching other components.

**Purpose:** lets users pause a commitment temporarily without losing history or breaking streaks. Covers travel, illness, Ramadan, or any planned break.

---

## Why Freeze Exists

Without freeze, a user traveling for two weeks has two options: miss every commitment (breaking streaks and hurting scores) or delete the commitment (losing history). Neither is acceptable. Freeze provides a third option: honest pause.

Frozen commitments are excluded from all scores and analysis — they don't count for or against the user. The rope doesn't decay. The streak doesn't break.

---

## Two Freeze Modes

**Indefinite** — paused until the user manually unfreezes. No end date. Use case: illness, open-ended break.

**Manual only** — user unfreezes when ready. No automatic resume. See `scheduled_freeze.md` for future scheduled freeze support.

---

## UI

Accessed via the ⋮ menu on any commitment card.

**Freeze dialog:**

```
Freeze "Morning Walk"?

This commitment will be paused.
History and streaks are preserved.

[ Freeze ]   [ Cancel ]
```

**Frozen card:** Shows ❄ icon. Tap ❄ → "Unfreeze now?" → confirm → resumes immediately.

Frozen commitments are hidden from the dashboard by default. Visible via the Frozen filter.

---

## Behavior While Frozen

- No new instances generated
- All pending instances within the freeze period get `status: frozen`
- Frozen instances excluded from Today Score, period scores, and all analytics
- Shown as neutral on calendar heatmap — not red, not green, just absent
- Streak pauses at current count — resumes from same count on unfreeze

---

## Service Logic

- Freeze → `CommitmentService.freezeCommitment(id)`
    
    - Sets `isFrozen: true` on definition
    - Sets `status: frozen` on all future pending instances
    - Stops instance generation for this commitment
- Unfreeze → `CommitmentService.unfreezeCommitment(id)`
    
    - Sets `isFrozen: false` on definition
    - Resumes instance generation from today
    - Streak count preserved — continues from where it paused

---

## Dependencies

- `CommitmentService` — freeze and unfreeze operations
- Presentation layer — watches `isFrozen` state via Riverpod to update card appearance