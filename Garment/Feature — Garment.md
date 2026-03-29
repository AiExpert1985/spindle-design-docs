**File Name**: feature_garment **Phase**: 2 (Accelerator) · 3 (Garment visuals) **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Garment gives each commitment a living visual form — a small garment that grows thread by thread with every kept day, or dissolves as a bad habit is broken. It makes the slow, invisible work of habit formation visible and personal.

---

## Why It Exists

Numbers don't make effort feel real. A streak count and a percentage are accurate — but they don't carry weight. People understand things they can see more than things they can count.

The garment is built on two truths. First, habits are formed day by day — like weaving, each kept day adds a thread. Skip a day and you add nothing. Keep going and something real takes shape. Second, a visual garment communicates that progress in a way no percentage can — its size, its fill, its completeness tells the story at a glance.

For Do commitments, the garment starts as a small seed and is woven toward completion with each successful day. For Avoid commitments, the garment is shown as visually complete from the start — representing the habit's current grip — and unravels as the user resists, back toward the bare spindle. The spindle is the origin of all garments — for Do, they grow away from it; for Avoid, they return to it. Commitment is the thread, consistency is the weaving, character is the rope.

When a garment completes, the journey does not end. The user keeps performing — fortifying the garment with layers of iron, then gold, then diamond. Each full cycle of continued effort adds the next layer. Without an upper bound, the garment keeps rewarding consistency indefinitely. A habit deeply embedded deserves to look different from one just starting.

---

## Position in the System

Sits above Performance, Streak, and Achievements. Depends on Performance for score events, Streak for momentum data, and Achievements to record milestones. Publishes `GarmentUpdatedEvent` for the presentation layer. No feature above Garment depends on it — it is a consumer and display feature.

---

## How It Works

When a commitment is created, Garment creates a profile for it — assigning a garment type based on the commitment's shape, and thread colors that become its permanent visual identity. From that point, the garment responds to every performance change through the day record, the accelerator, and the delta calculation.

---

### Day Records and Running Total

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

### Delta, Completion, and the Band System

Given the snapshotted acceleration and the day's performance, the delta calculator returns how much the garment moves. The delta is always positive and identical for Do and Avoid — the formula has no knowledge of commitment type. The contribution is proportional to the accelerated performance — a full kept day at 1.0 adds the maximum contribution, a partial day contributes proportionally, a missed day contributes zero. Missed days slow future progress through deceleration — the accelerator drops, making subsequent good days contribute less. What was accumulated stays accumulated.

**Delta calibration.** The base delta is set at 3.0% so that 21 consecutive days of full performance at the accelerator ceiling (1.5×) completes one garment cycle — roughly 4.5% × 21 days ≈ 94.5%. This makes near-perfect 21-day effort the intended completion time, reflecting the commonly cited habit-formation window. Configurable and will be tuned from real data after launch.

**Completion percent is unbounded above.** Capping at 100% would stop rewarding the user the moment a habit is formed — exactly when continued reinforcement matters most. Each 100% band represents one complete cycle:

- **0–100%** — the habit forming. The garment is being woven.
- **100–200%** — iron fortify. The habit is complete; the garment gains an iron layer.
- **200–300%** — gold. Mastery deepening.
- **300–400%** — diamond. A habit so embedded it has become identity.

The lower bound is 0.0 — a garment cannot go below empty. Decay — actively reducing `completionPercent` for missed days — is deferred: it conflicts with the app's honesty principle, risks frustrating users when they see progress shrink, and adds complexity. It may be revisited if real usage shows deceleration alone is insufficient.

All constants are configurable. The formula is abstracted — the entire calculation strategy can be replaced without touching `GarmentService`.

---

### Achievement Detection

After every update, the service checks whether the user has entered a new band using a simple formula: `floor(completionPercent / 100)`. If this value exceeds `currentLevel` on the profile, a new achievement fires and `currentLevel` is updated. One check, no list of thresholds, no risk of double-firing. Adding a new phase requires no logic change — only a new achievement subtype.

---

### Garment Identity

Each garment has a type reflecting the commitment's shape — simpler commitments get simpler garments, complex ones get more elaborate. Assigned once at creation, immutable. Thread colors are assigned once using seeded random selection from a curated vivid palette — the seed is the commitment's ID, so colors are always deterministic and stable across reinstalls. Both are part of the commitment's permanent visual identity.

---

### The Renderer — Where Do and Avoid Diverge

Everything below the renderer is commitment-type-agnostic. The delta formula, the day record, the accelerator, the running total, the band thresholds — all identical for Do and Avoid. `commitmentType` never enters any calculation.

The renderer is the single place where commitment type affects behavior. It receives `completionPercent` and `commitmentType` and decides what to draw:

**Do commitment** — the garment starts empty and grows. 0% is a bare seed. As `completionPercent` rises, threads are woven. At 100% the garment is complete. Beyond 100%, iron borders appear, then gold, then diamond — each band a visible layer of reinforcement on the completed form.

**Avoid commitment** — the garment starts as a full form (the habit's current grip, visually complete). As `completionPercent` rises, threads unravel from the form back toward the spindle. At 100% the spindle is bare — the habit is broken. Beyond 100%, the spindle itself gains adornment — an iron lock, then gold, then diamond — marking the habit as sealed away.

Same number. Opposite visual story. All complexity lives in one place.

The renderer is abstracted — `GarmentRenderer` is an interface with `SimpleThreadRenderer` as the Phase 1 implementation. The rendering strategy can be replaced without touching any calculation code.

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
- Achievement detection uses `floor(completionPercent / 100) > currentLevel` — simple, no threshold list, no double-firing
- The accelerator is internal — no feature outside Garment reads or writes it
- All constants in `AppConfig` — never hardcoded

---

## Later Improvements

**Decay mechanism.** Actively reduce `completionPercent` after sustained neglect. Deferred pending real usage evidence — the abstracted delta calculator makes it a zero-friction addition.

**Accelerator formula.** Structure is in place. Formula designed from real user data after launch.

**Asymmetric accelerator recovery.** Recovery faster than decline — separate up/down step constants in `AppConfig`.

**AI garment type assignment.** Pro/Premium — AI suggests type based on commitment shape. Rule-based always serves as synchronous fallback.

**User-chosen thread colors.** User picks from curated palette instead of seeded random. Phase 3.