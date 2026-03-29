**File Name**: feature_performance **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

# Feature — Performance

Performance is the score authority. It calculates how well the user is doing against their commitments, maintains a live score on every open instance, and answers any question about performance across any period. Nothing above it calculates scores independently.

---

## Why It Exists

Raw activity logs mean nothing without context. Ten pages logged is irrelevant without knowing the target was twenty, or that the commitment was weekly and the day's share was three. Performance owns the translation from raw logged values to meaningful scores — and it owns it in one place, so the rest of the system never has to think about scoring logic.

---

## Position in the System

Sits above Commitment and Activity, both of which it depends on directly. Anything above it that needs a score calls down into Performance. It is marked LOCKED — its scoring formulas and public interface are stable contracts that every feature above depends on.

Producing and consuming — it subscribes to events from Commitment and Activity, and publishes `PerformanceUpdatedEvent` after every score change.

---

## How It Works

Performance has two jobs: keeping scores current, and answering score queries.

**Keeping scores current.** Two events trigger a recalculation — a new or recreated instance, and any activity change. Both triggers exist for a clear reason: when the instance changes (target updated, instance recreated), the meaning of existing activity changes — reading 20 pages against a target of 20 is 100%, but against a new target of 40 it becomes 50%. When the user logs, edits, or deletes activity, the total changes and the score must follow. In both cases, Performance always recalculates from the current state — it never inspects what changed, just reads the current total and runs the formula.

Performance recalculates even for closed instances. If the user remembers to log activity two days later within the backfill window, the closed instance's score is updated to reflect what actually happened. Historical accuracy matters more than the convenience of treating closed instances as frozen.

**Per-instance performance — the core philosophy.** Each instance stores its own `livePerformance` as a raw percentage. This is the central design decision that gives the system its flexibility. To answer any performance question — one commitment this week, all commitments today, the full month — you simply filter the instances you want and average their scores. No special aggregation logic, no separate tables, no pre-computation. The average of instance scores tells the whole story. And because Do commitments can exceed 100%, the averaging is both accurate and meaningful — a week where the user overperformed on three days and underperformed on two averages to a fair reflection of their real effort.

**Writing back to the instance.** After calculating, Performance writes `livePerformance` directly onto the commitment instance via a direct downward call. This is intentional — it avoids a circular dependency where Commitment would need to subscribe to Performance events and Performance would depend on Commitment. Instead, Performance depends on Commitment, calls one narrow write function, and Commitment has no knowledge of Performance in return. The write is deliberately limited to `livePerformance` only — Performance cannot change any other field on the instance.

**No stored aggregates.** Weekly scores, period averages, day scores — none are pre-computed or stored. All calculated on demand by fetching instances and averaging their `livePerformance` values. Simple data model, no stale aggregate problem.

---

### The Scoring Formulas

**Do commitments** — `livePerformance = (totalLogged / target) × 100`. No upper cap. Logging above target records the real value — 150% means the user genuinely overperformed. This preserves real data and signals when a target may be too easy. A done/not-done commitment uses `target = 1` — logging 1 gives 100%.

**Avoid commitments** — success means staying at or below the target. Going over is failure, and the penalty should scale meaningfully with how far over. Three alternatives were evaluated:

- **Simple ratio** `(target / logged) × 100` — smoking 5 against a limit of 1 gives 20%. Compresses badly — a wide range of failures maps to a narrow band of low scores with no meaningful differentiation between minor and severe breaches.
- **Linear penalty** `max(0, 100 - excess × 100)` — floors at 0% the moment the user doubles the limit. Smoking 2 and smoking 20 both score 0%. All information beyond the first doubling is lost.
- **Pure exponential** — degrades too aggressively. 5× over gives near 0%. Meaningful differentiation disappears above 2× excess.

**Chosen: soft exponential with configurable base.** Every doubling of excess halves the score. The base is tunable — raising it toward 1.0 softens the penalty without touching the formula or any callers.

```
target == 0 and totalLogged == 0  →  100%   (stayed clean)
target == 0 and totalLogged > 0   →  0%     (any slip is failure)

target > 0 and totalLogged <= target  →  100%
target > 0 and totalLogged > target   →
  excess = (totalLogged - target) / target
  score  = 100 × AppConfig.avoidPenaltyBase ^ excess
```

|target|logged|excess|score (base 0.5)|
|---|---|---|---|
|1|1|0|100% — on target|
|1|2|1|50% — doubled the limit|
|1|3|2|25% — tripled|
|1|4|3|12.5% — four times over|
|3|4|0.33|79.4% — slightly over|
|3|6|1|50% — doubled|
|3|9|2|25% — tripled|

---

### Performance vs Success

Performance and success are distinct concepts and must not be confused.

**Performance** is a percentage — how much of the target was achieved. It is a continuous value that can be 47%, 100%, or 150%.

**Success** is a binary threshold — did the performance meet the minimum bar? A window with 85% performance is a failure if the success threshold is 90%. The threshold is configurable — `AppConfig.successThreshold` holds the default, and it may eventually become per-commitment.

`isWindowSuccess(livePerformance)` is the single function that applies this threshold. Every feature that needs to classify a result as kept or missed calls this function — it is never reimplemented elsewhere. Keeping this in one place means changing the threshold or making it per-commitment requires changing exactly one thing.

Target is used to calculate performance. Success threshold determines whether that performance was enough. The two are independent.

---

## Events

**`PerformanceUpdatedEvent`** — published after every `livePerformance` change. Carries the instance ID, definition ID, window start, and the new score.

The event is deliberately separate from `InstanceUpdatedEvent`. Structural instance changes and score changes serve different audiences — a feature that tracks live scores should not receive noise from recurrence changes, and a feature that reacts to window close should not receive every score tick.

---

## Who Uses It

Any feature that needs to evaluate whether a commitment window was kept calls `isWindowSuccess()`. Any feature that needs scores over a period — week totals, day scores, commitment history — calls the query functions. Features that need to react immediately when a score changes subscribe to `PerformanceUpdatedEvent`.

---

## Rules

- Two recalculation triggers only — `InstanceCreatedEvent` and `ActivityEvent`
- Writes `livePerformance` via a direct downward call to Commitment — limited to that one field only, no other instance field can be changed
- Never subscribes to events from features above it
- Recalculates only the affected instance per event — never bulk recalculation
- No stored aggregates — all scores calculated on demand
- `isWindowSuccess()` is the single definition of window success — never reimplemented elsewhere
- `livePerformance` stored as raw double — rounding is a display concern, never a storage concern

---

## Later Improvements

**Performance caching.** As history grows, period queries fetch potentially hundreds of instances. A lightweight cache keyed by period and `definitionId`, invalidated on `PerformanceUpdatedEvent`, would eliminate redundant fetches without changing the public interface. Phase 2.