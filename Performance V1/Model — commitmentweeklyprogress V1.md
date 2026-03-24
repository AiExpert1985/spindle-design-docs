
**Created**: 18-Mar-2026 
**Modified**: - 
**Feature**: Commitment 
**Phase**: 2

**Purpose:** stores the weekly garment progress delta for one commitment. One record per commitment per completed week. Used by the commitment detail screen to display the weekly progress table and the live current-week row.

---

## Fields

```
id: String                        // client-generated UUID
definitionId: String              // which commitment this belongs to
weekStart: DateTime               // Monday 00:00:00 of this week
weeklyDelta: double               // how much garmentCompletionPercent changed this week
                                  // positive = progress (weaving or unweaving)
                                  // negative = decay exceeded progress
                                  // stored, never re-derived
isCurrentWeek: bool               // true for the live in-progress week record
                                  // false once the week is complete and sealed
createdAt: DateTime
updatedAt: DateTime               // updated daily during the current week
```

---

## Two Record Types

**Completed week record** — written on Sunday night after the midnight garment update. `isCurrentWeek: false`. Never updated after creation. Represents the sealed delta for that Mon–Sun period.

**Live week record** — created on Monday morning when the week begins. `isCurrentWeek: true`. Updated every day by `GarmentCalculationService` as daily garment updates accumulate. Converted to a completed record on Sunday night.

---

## How weeklyDelta Is Calculated

`weeklyDelta` = `garmentCompletionPercent` at end of Sunday − `garmentCompletionPercent` at start of Monday.

For the live week record: `weeklyDelta` = current `garmentCompletionPercent` − value at start of Monday.

This means `weeklyDelta` naturally includes:

- Daily performance contributions
- Decay penalties if applicable
- Streak bonus contributions

All forces are captured in the single delta number. No breakdown is stored — the garment visual tells the full story.

---

## Display on Commitment Detail Screen

The weekly progress table reads these records ordered by `weekStart` descending — current week on top, oldest at bottom.

```
This week  (Mon–today)    +3%   ███░░░░░░░   ← live record, isCurrentWeek: true
Week of Mar 10            +5%   █████░░░░░
Week of Mar 3             +3%   ███░░░░░░░
Week of Feb 24            -1%   decay
Week of Feb 17            +7%   ███████░░░
```

Negative delta weeks show "decay" label — no bar, no color. Silence, not punishment.

Bar lengths are relative — the largest positive delta in the visible window gets full bar width. Others scale proportionally.

**First week behavior:** the table appears only after the first completed week. Before that, only the live week row is shown with the label: _"Your first week is still weaving."_

---

## Phase 3 Enhancement — Highlighted New Threads

When the garment display adds highlighted thread visualization (Phase 3), the `weeklyDelta` value for the current week is passed to the garment renderer to determine which threads to highlight as newly added. No additional field needed — `weeklyDelta` already carries this information.

---

## Rules

- One record per commitment per week — `definitionId + weekStart` is the unique key
- `weeklyDelta` is stored at calculation time, never re-derived
- Completed week records are immutable — never updated after `isCurrentWeek` is set to false
- Live week record is updated daily — `updatedAt` changes, `weekStart` never changes
- Frozen weeks produce no record — frozen commitments skip the daily garment update entirely
- Deleted or completed commitments retain their historical records — history is never erased

---

## Repository

Owned by `CommitmentRepository` — no separate repository needed. Add to the existing operations:

|Operation|Input|Output|
|---|---|---|
|`saveWeeklyProgress`|CommitmentWeeklyProgress|—|
|`fetchWeeklyProgress`|definitionId, limit?|List ordered by weekStart desc|
|`getLiveWeekProgress`|definitionId|CommitmentWeeklyProgress?|