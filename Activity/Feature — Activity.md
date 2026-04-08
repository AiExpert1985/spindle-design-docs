**File Name**: feature_activity **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Activity is where the user's effort is recorded. Every time the user logs something against a commitment — pages read, km walked, a cup of coffee avoided — that record lives here. It is the raw data source that every feature above it uses to measure, score, and reflect what actually happened.

---

## Why It Exists

Commitment defines what the user wants to do. Activity records what they actually did. Without this separation, the commitment model would need to carry performance data, making it responsible for both intent and reality — two concerns that must stay separate. Activity owns the raw log exclusively, and every feature that needs to know what happened reads from it.

---

## Position in the System

Sits above Commitment. Depends on Commitment only for cleanup — when a commitment is permanently deleted, Activity removes its own log entries. Otherwise fully independent: it receives write requests, validates them, writes, and publishes one event. It calls no other feature after writing.

Both producing and consuming — it publishes `ActivityEvent` after every write operation, and subscribes to one Commitment event for data cleanup.

---

## How It Works

The user logs an amount against a commitment for a specific date. Activity validates the date — only today and up to a configured number of past days are accepted, never future dates — and writes the entry. One event is published after every successful write, edit, or delete. All downstream reactions — performance recalculation, encouragement feedback — happen through subscriptions to that event. Activity itself calls nothing after writing.

**The backfill window.** Entries can be recorded or edited for today and up to `AppConfig.maxLogBackfillDays` in the past. This exists to allow reasonable corrections — forgetting to log while travelling, illness — while preventing unlimited retroactive changes that would silently corrupt scores and streaks. Attempts outside the window return a failure; the UI informs the user.

**Zero is never written.** A log entry only exists when the user recorded something. For Do commitments this means activity occurred. For Avoid commitments this means the user did the thing they were trying to avoid — the value is how much. Zero activity is represented by the absence of a log, not a zero-value entry. This keeps the model clean and queries simple.

**Activity logs belong to the definition, not the instance.** Entries carry `definitionId` and `loggedAt` only — no `instanceId`. This means instance lifecycle has no effect on activity history. If the user changes a target in the evening, the pending instance is cleared and a new one created — but the morning's logs remain correctly attributed to the commitment by date. The correct instance is identified at read time by finding the instance whose regeneration window contains `loggedAt`.

**What Activity does not do.** It never calculates scores, never evaluates whether a target was met, and never knows about streaks, garments, or progression. It records raw values and publishes them. Everything else is someone else's concern.

---

## Events

One event covers all three write operations — created, updated, deleted. A single event type is deliberate: any feature that needs to react to a change in activity subscribes to one stream and handles all three cases. Features that care only about new logs filter on the `created` type.

`value` is null on deleted — absence of a value signals removal without needing a separate event type.

---

## Who Uses It

Any feature that needs to know what the user logged on a given day or period reads from Activity. Features that react to new or changed activity subscribe to `ActivityEvent`. Any feature that needs to check whether activity exists for a commitment on a specific date calls `getTotalLoggedForCommitmentOnDate()` directly.

---

## Rules

- Logging allowed regardless of commitment state — frozen, completed, or active
- `loggedAt` is immutable after creation — moving a log to a different date is not allowed
- All write functions return `Result<T>` — failures are never silent
- All aggregations computed in Dart after fetching raw records — never delegated to the repository
- Bulk deletion only on permanent commitment deletion

---

## Later Improvements

**Context tags.** Predefined tags (tired, social, travel, motivated) attached at log time. Transforms raw activity data into behavioral data for Analytics and AI Insights to explain patterns. Requires adding `contextTags` to `LogEntry` and a tag selection UI. See `component_context_tags`.


---

## Related Docs

[[Model — Log Entry]]
[[P1 Component — Log Entry Dialog]]
[[P1 Component — Log History Sheet]]
[[P2 Component — Context Tags]]
[[Repository — Activity]]
[[Service — Activity]]
