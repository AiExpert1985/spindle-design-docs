**File Name**: feature_infrastructure **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Infrastructure provides three raw, domain-free capabilities — time signals, diagnostic output, and OS notification delivery — that any feature in the stack can use without knowing anything about the platform underneath.

---

## Why It Exists

Every feature needs at least one of: a way to react to time passing, a way to record what went wrong, a way to reach the user's notification tray. Without a shared foundation for each, every feature would depend directly on platform-specific packages. Infrastructure absorbs that dependency and exposes a stable, domain-free interface instead. Swapping the underlying platform requires changing one place — nothing above it is affected.

---

## Position in the System

Bottom of the dependency chain. Depends on nothing. Every feature above it may use any of its three capabilities freely. It never subscribes to events from any feature and never calls any feature's service — it has no knowledge that anything above it exists.

---

## How It Works

**Time signals.** The platform fires a periodic signal — short interval and long interval — that Heartbeat converts into a tick and broadcasts to any subscriber. Each tick carries only a raw timestamp. Subscribers decide for themselves what the timestamp means and what to do with it. When the app opens and the first tick fires, each subscriber processes everything due at or before that timestamp — missed intervals are caught up naturally with no special logic required.

**Diagnostic output.** Any feature that needs to record what happened — an error, a warning, a significant event — sends a message to Logger. Logger handles where it goes. In early phases, that means the developer console. Later it can mean persistent storage or a remote crash service. The rest of the app never changes.

**Notification delivery.** Any feature that needs to reach the user's notification tray sends a message and an optional destination route to NotificationService. It handles OS delivery. The caller composes the message — NotificationService never generates text. If permission is denied, delivery is dropped silently and logged.

---

## Who Uses It

**Time signals** are used by any feature that needs to react to time passing — checking whether a day has ended, whether a scheduled reminder is due, whether a weekly boundary has been crossed. The tick is the shared clock the whole system runs on.

**Logger** is used by any feature that needs to record a diagnostic event. It never produces anything user-visible.

**NotificationService** is used by any feature that needs to push a message to the user outside the app — scheduled reminders, weekly summaries, milestone alerts.

---

## Rules

- No Infrastructure service ever imports a domain model or calls a feature above it — domain knowledge never enters this layer
- Tick events carry a timestamp only — no pre-computed conclusions, no domain context
- Notification message text is always composed by the caller — never by NotificationService
- Every subscriber processes ticks as "everything due at or before this timestamp" — this is what makes app-open catch-up free

---

## Later Improvements

**Persistent logger.** Stores warnings and errors locally for post-session review. Phase 2.

**Remote logger.** Forwards errors to a crash reporting service. Requires user consent. Phase 3.

**Dual tick intervals.** Two separate platform tasks — one long, one short — for more accurate short-interval behavior. One implementation swap, zero subscriber changes. Phase 2.

**Remote push notifications.** A server-backed notification implementation for paid tiers. Phase 3.


---

## Related Docs

[[Service — Heartbeat]]
[[Service — Logger]]
[[Service — Notification]]
