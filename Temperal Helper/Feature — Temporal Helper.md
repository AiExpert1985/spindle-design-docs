**File Name**: feature_temporalhelper **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

TemporalHelper answers semantic questions about time using the current user's preferences, and publishes boundary events when a day or week ends or begins. It is the single place in the app where time is interpreted — no other feature does this independently.

---

## Why It Exists

A raw timestamp means nothing without context. Whether midnight is a day boundary depends on the user's configured day reset hour. Whether today is a rest day depends on the user's region. Whether a week has ended depends on the user's week start day. Without a centralized place to answer these questions, every time-sensitive feature would have to read user preferences itself and reimplement the same logic. TemporalHelper owns this responsibility once, and every feature above it calls down into it.

---

## Position in the System

Sits above UserCore and Infrastructure. Depends on both — it subscribes to the periodic time signal from Infrastructure and reads user preferences from UserCore. Every feature above it that needs to interpret time or react to a day or week boundary depends on it.

Producing and consuming — it subscribes to raw time signals from below and publishes meaningful boundary events upward.

---

## What It Owns

**Service:** `TemporalHelperService` — query functions for time interpretation and boundary event streams.

No model, no repository. TemporalHelper holds no persistent state — preferences are read from UserCore and cached in memory.

---

## How It Works

`TemporalHelperService` subscribes to the long-interval time signal. On each tick it checks whether a day or week boundary has been crossed, using the user's configured preferences. If a boundary is detected and not already published this session, it fires the appropriate events. Deduplication is handled in memory — the service tracks the last published day and week boundary and skips re-publishing within the same session.

Any feature that needs to answer a time question — is this a rest day, is this within waking hours, what is the start of the current week — calls a query function, passing a timestamp. The service answers using cached preferences. No caller needs to know anything about preferences or region.

**Key design decisions**

- Service, not a static utility — if it were static, every caller would have to read preferences itself before calling it. As a service it owns that dependency internally and caches it.
- Centralized boundary detection — one detection, one place. Features subscribe to a semantic signal rather than reimplementing boundary logic on raw ticks.
- In-memory deduplication only — boundary events fire at most once per session per boundary. A restart is acceptable — the first tick after restart re-fires any boundary that was missed, and all subscribers process it correctly because they handle "everything due at or before this timestamp."

**What TemporalHelper does not do:** it never knows about the features using it. It answers time questions and publishes time signals — nothing more.

---

## Who Uses It

Any feature that needs to react when a day ends, a day begins, a week ends, or a week begins subscribes to the appropriate boundary event. Any feature that needs to check whether a timestamp falls on a rest day, within waking hours, or at a boundary calls a query function. Features that need the start of the current or previous week for date-windowed queries call the week calculation functions.

---

## Rules

- No feature detects day or week boundaries independently — all boundary detection goes through this feature's events
- No feature calls `UserCoreService.getTemporalPreferences()` directly — `TemporalHelperService` owns that dependency
- Query functions are called with a timestamp — preferences handling is never the caller's concern
- No domain knowledge — never imports any model or event from features above it

---

## Later Improvements

No planned extensions for the current phase. The query interface may grow if new time-based questions emerge from features above — each new function is additive and requires no structural change.