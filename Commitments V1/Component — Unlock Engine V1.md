
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 2
**Standalone:** yes — can be added or removed without touching other components.

**Purpose:** controls when users can add new commitments. Prevents commitment overload — the primary cause of habit app abandonment. Applies to all tiers.

---

## Two Gates

Both must pass before a new commitment can be added.

**Gate 1 — Consistency** 7-day average score ≥ 50%. Frozen commitments excluded. Window extends backward automatically to always cover 7 active days.

**Gate 2 — Rate limit** Fewer than 3 new commitments added in the last 7 days. Rolling window. Deleted commitments do not count.

---

## Ceilings

|Tier|Max portfolio size|
|---|---|
|Free|7|
|Pro / Premium|No ceiling|

Portfolio size = active + frozen + completed + soft-deleted commitments. Permanent delete (hard delete) is the only action that frees a slot. Soft delete to recycle bin does NOT free a slot — the commitment can still be restored. Used by: `CommitmentService.getPortfolioSize()`

---

## Onboarding Bypass

First 3 commitments skip both gates. Users must be able to set up their initial commitments freely.

---

## UI — Add Button

The add commitment button (`[+ Spin]`) appears on the Threads screen and anywhere new commitments can be created.

- **Gates pass** → button is active
- **Any gate blocked** → button is dimmed, tapping opens the unlock dialog

---

## UI — Unlock Dialog

Shown when user taps the dimmed add button. Always shows specific numbers — never vague.

**Gate 1 blocked:**

```
Not yet — but you're close.

Keep 50% of your commitments for 7 days
to spin a new one.

Your consistency: 43% → Goal: 50%
████████░░░░  Est. ~4 days

Slow down — it's a feature, not a bug. 🙂

[ Got it ]
```

**Gate 2 blocked:**

```
Give it a few days.

You've added 3 commitments this week.
Adding more has a reverse effect —
weave before expanding.

Added this week: 3 of 3
Next slot available: Thursday

[ Got it ]
```

**Both gates blocked:** show Gate 1 message only — one problem at a time.

**Free tier ceiling reached, both gates green:**

```
You've mastered your threads.

You're at the free tier limit (7 commitments).
Upgrade to Pro to keep expanding.

[ Upgrade to Pro ]   [ Not now ]
```

---

## UI — Your Record (Section 1)

Passive summary only — no gate details here. Full details live in the dialog.

```
🟢  ⭐  🟠  🟢  🟢  ○  ○
5 active  ·  2 slots available

Free tier: 5 of 7  [ Upgrade to Pro ]
```

One colored dot per active commitment. Empty circles for available slots. Tap any commitment dot → navigates to Thread Detail for that commitment.

---

## Service Logic

**Evaluate gates** Reads last 7 days of commitment data. Computes average score. Counts new commitments added in the last 7 days. Returns gate status, current values, and estimated days to unlock. Pure read-only — writes nothing.

---

## Repository Operations

- Fetch commitment instances for last 7 days (read only)
- Fetch commitment definitions created in last 7 days (read only)

---

## Trigger

Evaluated in real time whenever the add button is rendered or tapped. No scheduler needed.

---

## Dependencies

- Commitment instances and definitions (read only)
- Subscription tier from user profile — to enforce ceiling per tier