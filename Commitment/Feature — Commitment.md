**File Name**: feature_commitment **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 30-Mar-2026

---

Commitment is the foundation of the app's domain. It is where the user defines what they want to commit to, and the system that keeps a continuous daily record of those commitments running — so every feature above it always has fresh, accurate data to work with.

---

## Why It Exists

The user's commitments are the central fact of the app. Every feature that evaluates performance, tracks progress, scores achievements, drives the garment, encourages the user, or schedules notifications needs to know what commitments exist, what their targets are, and whether their windows are open or closed. Commitment is the single place that owns this data and keeps it current. Without it, nothing above it can function.

---

## Position in the System

Sits above TemporalHelper and UserCore. Everything above it depends on it. It is marked LOCKED — its public interface is stable and any change to it is a breaking change for every feature above.

Commitment is both a producing and consuming feature. It subscribes to the periodic time signal to drive instance regeneration. It publishes instance events that form its entire public interface to the rest of the application.

---

## How It Works

The feature has two distinct parts.

**Part one — the definition.** This is what the user configured: the commitment's name, type (do or avoid), target, recurrence, activity window, and current lifecycle state. The definition is the user's intent. It exists primarily for the UI to display — upper features never consult it for operational decisions.

**Part two — the instances.** An instance is one daily occurrence of a commitment. Instances are what the rest of the system works with — performance is calculated against them, notifications are timed by them, streaks are built from them, the garment grows through them.

---

### The Regeneration Mechanism

The central design challenge is keeping a fresh pending instance available for every active commitment at all times. The solution is self-replication: each instance carries a DNA snapshot — a complete copy of everything it needs to spawn its successor. Name, type, target, recurrence, activity window, commitment state — all embedded in the instance itself. The definition is never consulted during normal replication.

This matters because it makes the system resilient and self-contained. A missed tick, a device restart, or a cold launch does not break the chain. The instance always has its own DNA. When its window closes, it produces the next instance immediately from that snapshot, in the same operation, before any other feature sees the closure event.

---

### One Instance Per Commitment at Any Time

A key design decision: there is always exactly one pending instance per commitment, regardless of recurrence type.

The alternative would be to pre-generate all future instances upfront — 7 daily instances spawning 7 children each, or specific-day instances repeating only on their own day. This creates a tree of pending instances that must be managed, invalidated, and rebuilt every time the user changes a target or recurrence. The complexity compounds quickly.

Instead, each commitment has exactly one pending instance at any moment. When it closes, it spawns exactly one successor — the next occurrence. A daily commitment's Monday instance spawns Tuesday. A specific-days commitment's Sunday instance spawns its next configured day (e.g. Friday). There is never more than one pending instance to clear, recreate, or reason about.

This is what makes definition changes trivially simple: when the user edits anything, there is exactly one pending instance to clear, and exactly one new instance to generate from the updated snapshot. Nothing else to find, nothing else to invalidate.

---

### Why All Instances Are Daily

Regardless of how a commitment is configured — daily, weekly, or specific days — all instances are daily.

A weekly commitment does not produce one weekly instance. It produces one instance per day, each carrying 1/7 of the weekly target. A weekly goal of 200 pages becomes a daily instance with a target of 28.6 pages.

**Why this matters:** it makes performance calculation uniform across the entire system. Every feature above Commitment treats every instance identically — there is no special-casing by recurrence type anywhere above this feature. The complexity of recurrence is absorbed entirely inside Commitment during instance generation and never leaks out.

It also enables flexible effort distribution across a week. A user who commits to walk 3 days a week gets a daily instance with 1/7 of the weekly target. On a day they walk, they may log far above the daily target — 300% or more — and that over-performance carries the week forward. This is what makes the 1/7 division mathematically sound: because over-performance is allowed and accumulates, a user who acts intensely on fewer days still reaches their weekly total correctly. The weekly picture self-corrects across days without any feature needing to know the commitment was configured as weekly.

**Partial first week.** A weekly commitment created mid-week does not wait for the next full week to start. It begins generating daily instances immediately. If the user's week ends on Friday and they add a weekly commitment on Thursday, they get 2 daily instances before the first `WeekEndedEvent`. Features that evaluate weekly performance — Streak, Cups — average across only the instances that exist, not across a full 7-day span. The user is judged on what they actually had. No special-casing or cycle-window concept is needed — the daily decomposition handles partial weeks naturally.

---

### The Two Windows

Every instance carries two distinct windows that serve different purposes and must never be confused.

**The regeneration window** is the accountability period — when this occurrence begins and ends. It drives instance close and replication timing. Immutable after creation. No effect on notifications.

**The activity window** is the user's preferred time of day. It governs notification timing only — when to send a warning, when to send a window-close reminder. No effect on scoring, no effect on when the instance closes, no effect on replication.

---

## Events

Commitment has two event layers — one internal, one public.

**Internal events** coordinate the two services inside the feature. Any change to the definition — creation, edit, or deletion — triggers the same response: the pending instance is cleared and recreated from the updated snapshot. The response is always identical regardless of what changed, which means zero special-case logic. Closed instances are never touched — they are permanent historical records. The only exception is `livePerformance`, which can be updated retroactively if the user logs activity after the fact. Closed instances are removed only when the commitment is permanently deleted.

**Public events** are the feature's entire contract with the rest of the system. Three events cover everything: a new instance was created, an instance changed (including window close), a commitment was permanently deleted. Every feature above subscribes to these events to do its work. None of them needs to read the definition directly.

---

## Rules

- `CommitmentEvent` is internal — never subscribed to outside this feature
- `commitmentType` is immutable after creation — changing it would make all historical performance meaningless
- Target and recurrence changes apply from today forward — closed instances are never modified
- State transitions only through `CommitmentService` functions — never by writing `commitmentState` directly
- The only public write function on instances is `updateLivePerformance()` — called by the performance feature only
- `livePerformance` changes do not trigger `InstanceUpdatedEvent` — they have their own signal
- Closed instances are permanent — deleted only on permanent commitment deletion
- Exactly one pending instance per active commitment at all times

---

## Later Improvements

No structural changes planned. The feature is LOCKED — extensions are additive only and require a full review of all consumers before merging.


---

## Related Docs

[[Model — Commitment Definition]]
[[Model — Commitment Instance]]
[[P1 Component — Commitment Card]]
[[P1 Component — Commitment Info]]
[[P1 Component — Commitment State Change]]
[[P2 Component — add commitment button]]
[[P3 Component — Adaptive Difficulty]]
[[P3 Component — Home Screen Widget]]
[[P3 Component — Scheduled Freeze]]
[[Repository — Commitment]]
[[Repository — Commitment Instance]]
[[Screen — Commitment Detail]]
[[Screen — Commitment Form]]
[[Screen — Commitments Main]]
[[Service — Commitment]]
[[Service — Commitment Identity]]

