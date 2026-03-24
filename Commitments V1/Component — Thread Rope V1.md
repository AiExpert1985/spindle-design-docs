**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Commitment **Phase**: 3 **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** visual representation of a commitment's current momentum on the dashboard card and overview screens. A rope made of colored strands — thicker means stronger habit, thinner means the habit still has a grip. Honest about current state, not just lifetime count.

---

## Scope — Two Visuals, Two Purposes

Spindle has two distinct commitment visuals. They must not be confused:

**Thread Rope (this component)** — shown on dashboard commitment cards and the commitments overview screen. Circular arc of braided strands. Represents momentum — how the habit is trending right now based on recent kept/missed days. Updates on every log event.

**Garment Display (`component_garment_display.md`)** — shown on the commitment detail screen only. A woven garment silhouette (sock, sweater, scarf, etc.). Represents long-term accumulation — how much total weaving work has been done over the full life of the commitment. Updates once per day at midnight.

Same metaphor, different data, different screens, different update cadence. Never place the garment on the dashboard. Never place the rope on the detail screen.

---

## The Core Idea

- **Do commitment** — rope starts thin and grows thicker with each kept day. Building a habit thread by thread.
- **Avoid commitment** — rope starts thick (the habit has a grip) and grows thinner with each successful avoided day. Unraveling it one thread at a time.

---

## Thread Count Rules

**Do commitments:**

- Starts at 3 threads (`doStartingThreads` — configurable)
- Each kept day → +1 thread
- Floor: never drops below 3 — potential always exists

**Avoid commitments:**

- Starts at 21 threads (`avoidStartingThreads` — configurable)
- Each successful avoided day → -1 thread
- Ceiling: never exceeds 21

---

## Miss Mechanic

A pattern of failure decays the rope — not a single bad day.

- Miss 1–2 → grace period, no penalty (`missGracePeriod: 2` — configurable)
- Miss 3 onward → each miss costs 1 thread, momentum accumulates
- Any single success → miss counter resets to 0, gain 1 thread immediately

```
Avoid thread example (starts at 21):
Miss 1  → counter: 1, no change     (21 threads)
Miss 2  → counter: 2, no change     (21 threads)
Miss 3  → counter: 3, lose 1        (20 threads)
Miss 4  → counter: 4, lose 1        (19 threads)
Success → counter: 0, gain 1        (20 threads)
Miss 1  → counter: 1, no change     (20 threads — fresh grace)
```

---

## Thread Colors

Each thread slot gets a random vivid color assigned once at commitment creation, stored permanently. Same strand always appears in the same color — making each rope feel personal.

Colors stored as a list of hex values on the commitment definition:

```
threadColors: List<String>    // e.g. ["#E84393", "#43C6E8", "#F5A623"]
```

These same colors are used by the Garment Display component — visual identity is consistent across both representations of the same commitment.

---

## UI — Dashboard Card (Phase 3)

Replaces the circular progress ring on dashboard commitment cards.

- Shown as a circular arc made of distinct colored braided strands
- Arc thickness reflects current thread count
- Adding a thread → new strand animates in and joins the braid
- Removing a thread → one strand frays and fades out
- Implemented as a custom Flutter canvas animation

```
        ╭────────╮
       ╱  ≋≋≋≋≋≋  ╲       ← braided colored strands
      │   72%       │
       ╲            ╱
        ╰────────╯
```

---

## UI — End-of-Day Gauge Enhancement (Phase 3)

Replaces the simple circular gauge in the end-of-day animation.

- All active commitment ropes animate simultaneously
- Threads weave in real time as the gauge sweeps to the day score
- Each commitment's rope shown as a colored strand in the combined arc
- Same threshold and message rules as the simple gauge (≥ 60% to fire)

This is a separate implementation from the dashboard rope — same metaphor, different data (all commitments vs one).

---

## Data Model

Fields on `CommitmentDefinition` (added in Phase 3, null in Phase 1 and 2):

```
threadCount: int              // current number of threads
consecutiveMissCount: int     // resets to 0 on any success
threadColors: List<String>    // hex colors, assigned once at creation, never changed
```

Note: `consecutiveMissCount` here is for the rope miss mechanic. It is separate from `consecutiveFailDaysForDecay` which belongs to the garment calculation system. Both track consecutive failures but serve different systems with different rules.

---

## Service Logic

Stateless pure function — given current state and log result, returns new state. No side effects.

**Inputs:** `threadCount`, `consecutiveMissCount`, `commitmentType`, `logResult (kept | missed)`

**Output:** updated `threadCount`, `consecutiveMissCount`

**Logic:**

- Kept → reset miss counter to 0, increment or decrement thread count based on type
- Missed → increment miss counter, if counter > grace period then apply thread penalty
- Always respect floor (do: 3) and ceiling (avoid: 21)

**Configurable constants:**

- `doStartingThreads` — default 3
- `avoidStartingThreads` — default 21
- `missGracePeriod` — default 2

---

## Repository Operations

- Calculates new thread count and miss counter internally
- Saves updated definition via `CommitmentService.saveDefinitionState()` — not `updateCommitment()`, so version is not incremented

---

## Trigger

Runs automatically on every commitment log event — same trigger as streaks, via `PostLogOrchestrator`.

---

## Dependencies

- `PostLogOrchestrator` — triggers rope update after every log
- `CommitmentService.saveDefinitionState()` — persists updated thread fields
- Presentation layer — watches thread count via Riverpod to trigger strand animation