
**Created**: 18-Mar-2026 **Modified**: - **Feature**: Performance **Phase**: 2

**Purpose:** records the net performance change for one commitment for one completed window. Append-only — never edited or deleted. The single source of truth for all performance calculations: garment completion, weekly cup ratio, and streak contribution. Every percentage point in the garment can be traced to a specific entry.

---

## Fields

```
id: String                    // client-generated UUID
definitionId: String          // which commitment this belongs to
windowStart: DateTime         // start of the commitment window this entry covers
windowEnd: DateTime           // end of the commitment window this entry covers
recurrenceType: String        // snapshot of recurrenceType at time of entry
                              // stored so historical entries remain correct
                              // if recurrence is later changed

// Component contributions — stored separately for auditability
baseContribution: double      // 100 / targetRepetitions for this recurrenceType
                              // e.g. daily: 3.33%, weekly: 5.0%, monthly: 8.33%
                              // scaled by progressPercent of the instance
                              // 0.0 if instance was missed

decayPenalty: double          // negative value — applied when consecutiveFailWindows >= decayThreshold
                              // 0.0 if no decay this window
                              // e.g. -1.0

streakBonus: double           // positive value — applied when consecutiveKeptWindows >= streakBonusThreshold
                              // 0.0 if no streak bonus this window
                              // e.g. +1.0

performanceBonus: double      // positive value — awarded for exceeding target
                              // 0.0 for binary commitments or exact target hit
                              // tiered: slight exceed → +1.0, strong exceed → +2.0, exceptional → +3.0
                              // see PerformanceAccountingService for exact thresholds

netChange: double             // sum of all components above — the single value used by consumers
                              // clamped so garmentCompletionPercent never exceeds 100.0 or drops below 0.0
                              // stored after clamping — always reflects what was actually applied

createdAt: DateTime           // when this entry was written — always the window close time
```

---

## Append-Only Rule

Entries are never edited or deleted. If a window is re-evaluated (e.g. after a late grace period log), a correction entry is written with the delta — not an edit of the original. This preserves the complete audit trail.

Exception: when a commitment is permanently deleted, all its entries are deleted as part of the cascade. This is acceptable — the commitment no longer exists.

---

## One Entry Per Window Close

One entry is written per commitment per window close, regardless of recurrence type:

- Daily commitment → one entry per day at midnight
- Weekly commitment → one entry per week on Sunday night
- Monthly commitment → one entry per month at month end
- Custom commitment → one entry per custom window close

This means a weekly commitment's single entry can have a larger `baseContribution` than a daily commitment's entry — because `targetRepetitions` for weekly is 20, giving `100/20 = 5.0%` per kept window vs `100/30 = 3.33%` for daily. The system is fair across recurrence types by design.

---

## Performance Bonus Tiers

For measurable (non-binary) commitments, performance above target earns bonus contribution:

|Achievement|Bonus|
|---|---|
|Target met exactly (100%)|+0.0|
|Exceeded by 10–29%|+1.0|
|Exceeded by 30–59%|+2.0|
|Exceeded by 60%+|+3.0|

Binary commitments always receive +0.0 performance bonus — done is done. Avoid commitments: staying well under the limit earns a bonus using the same tier logic in reverse (further under limit = higher bonus).

---

## Frozen Windows

No entry is written for frozen commitment windows. Frozen periods produce no entries — they are honest pauses, not failures. The garment does not decay during freeze, and the weekly cup calculation excludes frozen commitments from both numerator and denominator automatically.

---

## Rules

- Append-only — never edit, never delete except on permanent commitment deletion
- One entry per window close per commitment — never multiple entries for the same window
- `netChange` is always stored clamped — the raw sum is not stored separately
- `recurrenceType` is snapshotted at entry time — historical entries are never retroactively altered
- Frozen windows produce no entry — absence of entry = frozen period
- All component fields (`baseContribution`, `decayPenalty`, `streakBonus`, `performanceBonus`) are stored even when zero — for auditability and future analytics