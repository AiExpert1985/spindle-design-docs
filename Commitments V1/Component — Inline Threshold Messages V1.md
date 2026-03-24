
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase**: 3
**Standalone:** yes — purely additive, zero changes to existing logging or reward components.

**Purpose:** short contextual messages that appear during a logging session at score milestones. Reinforces progress without requiring the user to look away from the card they just logged.

---

## When They Appear

Shown inline below the commitment card after a log, fading after 2 seconds. Only at meaningful crossings — never on every log.

|Trigger|Message|
|---|---|
|First log of the day|"Good start."|
|Commitment reaches 50%|"Halfway there."|
|Today Score crosses 80%|"You're on track."|
|Commitment reaches 100%|"Commitment kept."|

---

## Rules

- Fade after 2 seconds — never blocks interaction
- Never shown on misses, breaches, or "I slipped"
- One message per crossing — never repeats in the same session
- Silent if the animation system is under load — see UX principles

---

## Dependencies

- Presentation layer only — watches instance state via Riverpod
- No service calls, no storage