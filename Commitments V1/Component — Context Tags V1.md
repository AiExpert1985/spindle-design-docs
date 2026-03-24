
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 2 (UI) — data fields exist in Phase 1 models

**Purpose:** captures why something happened at key logging moments. Transforms raw activity data into behavioral data — enabling the analytics and AI features to explain patterns rather than just describe them.

---

## Why Tags Matter

Without tags, analytics can only say: "You exceed your coffee limit in the afternoon." With tags, analytics can say: "You exceed your coffee limit mostly when tagged 'tired', on workdays between 3–6pm."

Tags answer the question that timestamps alone cannot.

---

## Where Tags Are Stored

```
LogEntry.contextTags: List<String>         // why this specific log happened
CommitmentInstance.contextTags: List<String>  // why this instance went well or poorly overall
```

Both fields exist on the models from Phase 1 — Phase 2 adds the UI that populates them.

---

## When Tag Sheet Appears

Only at notable moments — never during normal successful logging:

|Moment|Sheet shown|Tags stored on|
|---|---|---|
|Avoid commitment exceeded|Immediately after log|LogEntry|
|"I slipped" on zero-tolerance|Immediately|LogEntry|
|Instance missed (swipe left)|Inside failure recovery sheet|CommitmentInstance|
|Overachievement (well above target)|Optional prompt|CommitmentInstance|

---

## Tag Sheet UI

```
What happened?

[ tired ]  [ stressed ]  [ social ]  [ bored ]  [ work break ]

+ Add your own

[ Done ]
```

One tap for predefined tags. Keyboard for custom tags. "Done" dismisses — tags are optional, never required.

---

## Predefined Tag Sets

**Avoid commitments** (coffee, alcohol, smoking): `tired | stressed | social | bored | work break`

**Do commitments missed** (exercise, reading): `busy | forgot | low energy | schedule conflict | traveling`

Predefined sets are commitment-type specific — the right tags appear automatically.

---

## Energy / Mood Tag

Optional 1–5 energy rating shown as emoji scale at any log moment.

```
How are you feeling?
😴  😕  😐  🙂  ⚡
 1   2   3   4   5
```

Stored as `energyLevel: int?` on `CommitmentInstance`. Enables cross-variable insights: "You miss exercise when energy ≤ 2."

---

## Silent Context (No User Action)

These fields are stored automatically on every instance at creation — no user interaction needed:

```
dayOfWeek: int    // 1–7
hourOfDay: int    // 0–23
isWeekend: bool
```

Gives analytics and AI structured temporal context for free. Already documented in `commitment_instance_model.md`.

---

## Service Logic

- Tag sheet confirmed → `CommitmentService.updateInstance(updatedInstance)`
- Energy rating selected → `CommitmentService.updateInstance(updatedInstance)`
- Log entry tags → written directly via `CommitmentService.logEntry()` or `logFailureRecovery()`

---

## Dependencies

- `CommitmentService` — writes tags to instances and log entries
- Commitment logging component — tag sheet is triggered by specific logging moments
- `CommitmentAnalyticsService` — reads tags for pattern analysis (Phase 2)