**File Name**: component_context_tags **Feature**: Activity **Phase**: 2 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** captures why something happened at key logging moments. Transforms raw activity data into behavioral data — enabling Analytics and AI Insights to explain patterns rather than just describe them.

Without tags, analytics can only say: "You exceed your coffee limit in the afternoon." With tags: "You exceed your coffee limit mostly when tagged 'tired', on workdays between 3–6pm." Tags answer the question that timestamps alone cannot.

Expected placement: shown as an optional step inside the log entry dialog, at specific moments described below.

---

## When Tags Are Offered

Only at notable moments — never during normal successful logging.

|Moment|Tags offered|
|---|---|
|Avoid commitment exceeded|After the log is recorded|
|Zero-tolerance avoid ("I slipped")|After the log is recorded|
|Do commitment logged below target at window close|After window closes|

Tags are always optional. Dismissing without selecting any is fine — the log is already recorded at this point.

---

## UI

```
What happened?

[ tired ]  [ stressed ]  [ social ]  [ bored ]  [ work break ]

+ Add your own

[ Done ]
```

One tap selects a predefined tag. Multiple tags can be selected. "Add your own" opens a text field for a custom tag. "Done" saves and dismisses.

---

## Predefined Tag Sets

Tag set shown depends on commitment type — the right tags appear automatically.

**Avoid commitments** (coffee, alcohol, smoking): `tired · stressed · social · bored · work break`

**Do commitments** (exercise, reading, any habit): `busy · forgot · low energy · schedule conflict · traveling`

---

## Storage

Tags written to `LogEntry.contextTags` — a list of strings on the log entry for that moment. See `model_log_entry`.

---

## Data Sources

|Data|Source|
|---|---|
|Write tags|`ActivityService.editEntry(id, contextTags)` — updates the log entry just recorded|

---

## Later Improvements

**Energy / mood rating.** An optional 1–5 energy scale shown alongside the tag sheet. Stored as a field on `LogEntry`. Enables cross-variable insights: "You miss exercise when energy ≤ 2." Requires a model addition to `LogEntry`.

**Overachievement tagging.** Tag prompt when a Do commitment is logged well above target — captures what went right, not just what went wrong.