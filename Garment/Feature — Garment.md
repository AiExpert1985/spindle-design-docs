**File Name**: feature_garment **Phase**: 2 (Accelerator) · 3 (Garment visuals) **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

# Feature — Garment

Garment gives each commitment a living visual form — a small garment that grows thread by thread with every kept day, or dissolves as a bad habit is broken. It makes the slow, invisible work of habit formation visible and personal.

---

## Why It Exists

Numbers don't make effort feel real. A streak count and a percentage are accurate — but they don't carry weight. People understand things they can see more than things they can count.

The garment is built on two truths. First, habits are formed day by day — like weaving, each kept day adds a thread. Skip a day and you add nothing. Keep going and something real takes shape. Second, a visual garment communicates that progress in a way no percentage can — its size, its fill, its completeness tells the story at a glance.

For Do commitments, the garment starts as a small seed and is woven toward completion with each successful day. For Avoid commitments, it starts complete and slowly unravels as the user resists — back toward the spindle. The spindle is where all garments begin. Commitment is the thread, consistency is the weaving, character is the rope.

When a garment completes, the journey does not end. The user keeps performing — fortifying the garment with layers of iron, then gold, then diamond. Each full cycle of continued effort adds the next layer. This is intentional: without an upper bound, the garment keeps rewarding consistency indefinitely. A habit deeply embedded deserves to look different from one just starting.

---

## Position in the System

Sits above Performance, Streak, and Achievements. Depends on Performance for score events, Streak for momentum data, and Achievements to record milestones. Publishes `GarmentUpdatedEvent` for the presentation layer. No feature above Garment depends on it — it is a consumer and display feature.

---

## How It Works

When a commitment is created, Garment creates a profile for it — assigning a garment type based on the commitment's shape, and thread colors that become its permanent visual identity. From that point, the garment responds to every performance change through three concepts: the day record, the accelerator, and the delta.

---

### Completion Percent and the Band System

`completionPercent` is the running sum of all daily deltas. It has no upper bound — this is a deliberate design decision. Capping at 100% would stop rewarding the user the moment a habit is formed, which is exactly when continued reinforcement matters most.

Each 100% band represents one complete cycle of effort:

- **0–100%** — the habit forming. The garment is being woven.
- **100–200%** — iron fortify. The habit is complete; the garment gains an iron reinforcement layer.
- **200–300%** — gold fortify. Mastery deepening.
- **300–400%** — diamond. A habit so embedded it has become identity.

**Delta calibration.** The base delta (`dailyFullContribution`) is set at 3.0% so that 21 consecutive days of full performance at the accelerator ceiling (1.5×) completes one garment cycle — roughly 4.5% × 21 days ≈ 94.5%. This makes 21 days of near-perfect effort the intended completion time, reflecting the commonly cited habit-formation window. The value is configurable and will be tuned from real data after launch.

The lower bound is 0.0 — a garment cannot go below empty. There is no upper bound.

Every active commitment day has one `GarmentDayRecord` — created when the day's instance opens. Each record carries the date, an acceleration value (snapshotted at creation, immutable), and a delta (starts at zero, recalculates on any performance change for that day).

`GarmentProfile.completionPercent` is a running total. On any recalculation, the old delta is subtracted and the new one added — one arithmetic operation regardless of history length.

**Why not a simple running total.** A single number updated live has silent correctness bugs: a second log in the same day is blocked by the daily-update guard, a retroactive log within the backfill window cannot update an already-processed day, and a mid-day target change recreates the instance but the garment update is blocked. In all three cases the garment silently reflects the wrong state. The per-day record eliminates all three — any change to a day triggers a recalculation of that day's delta only, and the running total updates in one step.

**Why not sum all records on every read.** Commitments are bounded in length. Summing is not expensive — but it is unnecessary work on every screen open. The running total avoids it.

---

### The Accelerator

Consistency compounds. Reading every day builds the habit faster than reading one day and missing the next — the garment should reflect that. The accelerator is the mechanism that makes it real.

It is a single multiplier per commitment. It starts at 1.0 (neutral) and moves in both directions based on the commitment's signed streak — positive means consecutive successful days, negative means consecutive failed days. Each new streak day steps the multiplier toward the ceiling or floor following a configurable function. It is bounded: never above `acceleratorCeiling` (default: 1.5) or below `acceleratorFloor` (default: 0.75).

The accelerator's value is snapshotted into the day's record at record creation — when the day opens. That snapshot is immutable for the life of the record. Today's performance affects tomorrow's acceleration, never today's. This makes each day record fully self-contained: no matter how many logs fire during the day or how late a retroactive log arrives, the acceleration used for that day's delta never changes.

The accelerator updates live on each performance event — but its effect on the garment is always through the immutable snapshot in the day record, not through the live value.

The accelerator is optional. `AppConfig.garmentUsesAccelerator` is a master switch — when off, `getMultiplier()` always returns 1.0 and all deltas are calculated from raw performance with no momentum effect.

---

### Delta Calculation

Given the snapshotted acceleration and the day's performance, the delta calculator returns how much the garment moves. The contribution is proportional to the accelerated performance — a full kept day at 1.0 adds the maximum contribution, a partial day contributes proportionally, a missed day contributes zero.

Missed days slow future progress through deceleration — the accelerator drops, making subsequent good days contribute less. What was woven stays woven. Decay — actively reducing `completionPercent` for missed days — is deferred: it conflicts with the app's honesty principle (past effort happened and should not be erased), risks frustrating users when they see progress visibly shrink, and adds implementation complexity. It may be revisited if real usage shows deceleration alone is insufficient.

All constants are configurable and will be tuned from real data. The formula is abstracted — the entire calculation strategy can be replaced without touching `GarmentService`.

---

### Achievement Detection

After every update, the service checks whether a meaningful threshold was crossed — reaching 50%, completing the garment, completing the fortify phase, completing the gold phase. Detection fires only on genuine crossings. Achievement records are written to the Achievements feature below Garment.

---

### Garment Identity

Each garment has a type reflecting the commitment's shape — simpler commitments get simpler garments, complex ones get more elaborate. Assigned once at creation, immutable. Thread colors are assigned once using seeded random selection from a curated vivid palette — the seed is the commitment's ID, so colors are always deterministic and stable across reinstalls. Both are part of the commitment's permanent visual identity.

For Do commitments, the garment grows with consistency. For Avoid commitments, the direction reverses — it begins complete and unravels as the user resists. All calculations work identically — only the direction flips.

---

## Events

**`GarmentUpdatedEvent`** — published after every performance change, no matter how small. The garment is always live — the user's most recent activity is always reflected immediately. There is no batch computation, no daily summary, no delay. Log activity and the garment moves. Carries `definitionId` and `completionPercent`. Consumed by the presentation layer only.

---

## Rules

- Garment fields never appear on `CommitmentDefinition` — all garment state is owned here
- Garment type and thread colors assigned once at creation — never changed
- `GarmentDayRecord.accelerationValue` is immutable — snapshotted at day open, never updated
- `completionPercent` maintained as a running total via subtract-add — never summed from all records
- `completionPercent` has no upper bound — lower bound is 0.0 only
- Achievement detection fires only on genuine upward threshold crossings (100%, 200%, 300%, 400%)
- The accelerator is internal — no feature outside Garment reads or writes it
- All constants in `AppConfig` — never hardcoded

---

## Later Improvements

**Decay mechanism.** Actively reduce `completionPercent` after sustained neglect. Deferred pending real usage evidence — the abstracted delta calculator makes it a zero-friction addition.

**Accelerator formula.** Structure is in place. Formula designed from real user data after launch.

**Asymmetric accelerator recovery.** Recovery faster than decline — separate up/down step constants in `AppConfig`.

**AI garment type assignment.** Pro/Premium — AI suggests type based on commitment shape. Rule-based always serves as synchronous fallback.

**User-chosen thread colors.** User picks from curated palette instead of seeded random. Phase 3.