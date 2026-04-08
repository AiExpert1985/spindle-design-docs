**File Name**: feature_core **Feature**: Core **Phase**: 1 (partial) · Phase 2 · Phase 3 **Created**: 08-Apr-2026 **Modified**: 08-Apr-2026

---

Core is the presentation layer of the app — the screens, generic UI components, and system-wide configuration that assemble everything the user sees and interacts with.

---

## Why It Exists

Every feature in the system produces data, events, and service functions. None of them produce screens. Core is the layer that takes all of that output and assembles it into a coherent user experience — the daily logging surface, the progress record, the navigation structure, and the visual vocabulary. It also owns `AppConfig`, the single source of truth for every compile-time constant in the system.

Core has no business logic of its own. It reads from features below it, renders what they produce, and forwards user intent back down to services. If Core were removed, the entire system would still function — it just wouldn't be visible.

---

## Position in the System

Sits at the top of the dependency chain — above all features. It depends on everything below it but nothing depends on it. It calls services, watches providers, and renders state. It never owns data, publishes events, or makes decisions.

`AppConfig` is the one exception to "no logic" — it is a compile-time constants class owned by Core that every feature at every level references. It has no dependencies and no runtime behavior.

---

## What It Owns

**Screens** — the navigable destinations in the app. Each screen assembles components from one or more features into a coherent view.

**Generic UI components** — reusable visual elements with no feature knowledge. They receive data from their parent and render it. `ScoreRing` is the primary example — a circular arc widget that renders any ratio, used in multiple contexts across the app.

**`AppConfig`** — all system-level compile-time constants. Scoring thresholds, garment parameters, streak intervals, notification timing, progression point values, cup thresholds. No service or repository hardcodes these values — they always reference `AppConfig`. See `appconfiguration` for the full list.

**`Error Handling`** — the `Result<T>` type and `AppError` used across every feature boundary. Defined here in the Domain layer so every feature can reference it without creating a dependency on any other feature. See `error_handling`.

---

## How It Works

Screens are passive assemblers. A screen opens a set of providers, passes data down to components, and routes user actions back to services. No screen contains business logic — conditions that represent rules live in services, not in `if` statements in widgets.

Generic components receive everything they need from their parent. They never fetch data independently, never subscribe to events directly, and never call services. The `ScoreRing` receives a `double` and renders an arc. The parent decides what that double means and where it came from.

`AppConfig` is a pure static class — no instantiation, no runtime loading, no database. Every value is a compile-time constant. Changing a value here changes it everywhere that references it. User-configurable preferences are not here — they live in `UserCoreProfile` and are managed by the settings feature.

---

## Screens

**Dashboard** — the user's daily home base and the primary logging surface. Opens on app launch. Shows today's commitments as interactive cards. Performance summary above the cards gives an at-a-glance read on today and this week. See `screen_dashboard`.

**Your Record** — the user's complete progress picture in one scroll. A reflective screen opened a few times a week — not daily. Shows commitment health, performance history, streaks, cups, and progression across all commitments. Not in bottom nav — accessed via deep-links from notifications and other screens. See `screen_your_record`.

Other screens — Commitments, Commitment Detail, Commitment Form, Settings, Recycle Bin, Progression, Achievements, Referral — are owned by their respective features. Core owns only the screens that serve the whole-app view: the daily home and the long-term record.

---

## Generic Components

**Score Ring** (`component_score_ring`) — displays any ratio as a circular arc with sweep animation. Receives a value (0.0–1.0) and an optional label. Used on the Dashboard for overall today and weekly scores, on Commitment Detail for per-commitment scores, and on the Day Celebration screen. Zero changes needed to add new placements.

**Share Progress** (`component_share_progress`) — captures a progress snapshot as an image and invokes the native share sheet. Two templates: today snapshot (Template A) and weekly summary (Template B). Free for all tiers. Used on the Day Celebration screen and Your Record screen. See `component_share_progress`.

---

## Who Uses It

Core is the top of the chain — nothing depends on it. The presentation layer consumes it directly. Users interact with Core screens daily.

---

## Related Docs:

[[App Configuration]]
[[Error Handling]]
[[P2 Component — Score Ring]]
[[P3 Component — Share Progress]]
[[Screen — Dashboard]]
[[Screen — Your Record]]