**File Name**: feature_encouragement **Phase**: 2 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

Encouragement is the app's emotional voice for the small, unrecorded moments. It watches activity, performance, and commitment patterns — and responds when the system notices something worth acknowledging: a strong day, a comeback after struggle, partial effort that still counts. These moments are never stored as achievements. They are ephemeral human signals that make the app feel like it is paying attention.

---

## Why It Exists

Achievements record what the user has earned. Encouragement notices what the user is doing — the behavioral texture that falls between formal milestones. A user who kept all their commitments today earns no achievement for it — but the app should notice. A user returning after several missed days has crossed no threshold — but the moment deserves acknowledgment.

Encouragement is the only place in the app that produces this kind of dynamic display copy. No other feature knows what to say to the user about their day.

---

## Position in the System

Sits above Activity, Performance, Commitment, UserCore, and UserSettings — reads from all of them. Produces no data any other feature depends on. Pure consumer and broadcaster. Nothing below it depends on it.

---

## How It Works

Encouragement is purely reactive. It subscribes to events, reads context from below, selects a response, and emits an ephemeral signal via a Riverpod provider. The presentation layer observes the provider and renders accordingly. Nothing is stored — once a signal is consumed it is gone.

The one exception is `lastEncouragementType` on `UserSettingsProfile` — written through `UserSettingsService` to prevent the same celebration story from repeating on consecutive days.

### Three Trigger Paths

**1. Log feedback** — fires immediately when the user logs activity and performance increases. Emits an inline `LogFeedbackSignal` showing the new performance value. Also detects threshold crossings (50%, 100%) in memory and emits a `ThresholdMessageSignal` once per crossing per session — not stored, fire-and-forget. Deferred to Phase 2 when the Dashboard interaction is designed.

**2. Day celebration (Path A — in-app)** — fires when the last open window for today closes and the day score meets the configured threshold. Selects a story type from seven options based on today's behavioral context, never repeating the same type two consecutive days. Emits a `DayCelebrationSignal`.

**3. Day celebration (Path B — scheduled)** — fires at the user's configured celebration time via the Heartbeat tick. Same logic as Path A but triggered by time rather than the last window closing. Exactly one path fires per day — guarded by an in-memory flag.

### Day Celebration Story Selection

Seven story types, selected by priority. Type 6 (overall score) always applies as fallback. Never repeats the same type on consecutive days.

|Priority|Story Type|Example|
|---|---|---|
|1|Comeback|"After a tough few days, you're back at 71%."|
|2|Improvement over yesterday|"Up from 61% to 74%. Moving forward."|
|3|Commitment spotlight|"You walked every day this week."|
|4|Consistency (3+ weeks)|"Three weeks of quiet discipline."|
|5|Partial progress|"3km of your 5km walk. Partial progress moves you forward."|
|6|Overall score (fallback)|"83% today. Your garment grew stronger."|
|7|Slight setback (still ≥ threshold)|"Slightly below yesterday, but 65% still counts."|

Story context — whether the user is on a comeback, improving, consistent, or struggling — is derived directly from Performance and Activity reads. No separate analytics feature needed.

---

## Signals

All signals are ephemeral — emitted via Riverpod providers, consumed once by the presentation layer, and cleared. The presentation layer observes providers directly — never subscribes to events.

```
LogFeedbackSignal       — inline log reaction (Phase 2)
ThresholdMessageSignal  — 50% or 100% crossing (Phase 2)
DayCelebrationSignal    — end-of-day story
```

---

## Presentation Components

Three Phase 3 components consume Encouragement's signals. All are purely presentational — they render what the signal carries, never select or compute.

**Log Reward Animation** — instant visual feedback on every positive log. A value label (`+5 pages`, `✓`) floats up from the tap point and fades. Fires on any log that moves a commitment forward. Purely ephemeral — plays once and is gone. Observes `LogFeedbackSignal`.

**Inline Threshold Messages** — short contextual messages at meaningful score crossings (first log of the day, 50%, 100% on a commitment, 80% overall). Appear inline below the commitment card, fade after 2 seconds, never block interaction. Fire once per crossing per session — deduplication owned by `EncouragementService`. Observes `ThresholdMessageSignal`.

**Day Celebration** — full-screen ephemeral overlay when the day score meets the configured threshold. Shows the score, a short score-to-message label, and the story text selected by `EncouragementService`. Auto-dismisses after ~3 seconds and navigates to the Dashboard. Never fires on bad days — threshold check owned by the service. Observes `DayCelebrationSignal`.

All three are Phase 2 deferred — pending the Dashboard card interaction design.

---

## Rules

- No persistent storage — `lastEncouragementType` is the only written state, via `UserSettingsService`
- Day celebration fires at most once per day — in-memory guard prevents both paths firing
- Threshold crossings tracked in memory — fire once per crossing per session, never stored
- Signals are ephemeral — never replayed or stored
- Display copy lives here — not in any producing feature or event
- Presentation layer observes Riverpod providers — never subscribes to events directly
- Does not react to achievement or level-up events — those moments are handled elsewhere
- `LogFeedbackSignal` and `ThresholdMessageSignal` deferred to Phase 2

---

## Later Improvements

**Log feedback and threshold messages (Phase 2).** Deferred until the Dashboard card interaction is designed.

**Richer story types.** More story types can be added by extending the priority table. No structural changes needed — story context is already read from Performance and Activity on every evaluation.