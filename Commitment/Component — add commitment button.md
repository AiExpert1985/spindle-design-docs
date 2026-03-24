**File Name**: component_add_commitment_button **Feature**: Commitment **Phase**: 1 (button only) · Phase 2 (gate enforcement) **Created**: 23-Mar-2026 **Modified**: 23-Mar-2026

---

**Purpose:** the Add button that opens the Commitment Form. In Phase 1 it is always active. From Phase 2 it enforces access gates by observing `UserCapabilityService` — dimming the button and showing a contextual dialog when the user is blocked. Contains no gate logic itself.

Expected placement: bottom of any screen where new commitments can be created — currently the Commitments screen.

---

## Phase 1 Behavior

Button is always active. Tap → opens Commitment Form in create mode. No gate check.

---

## Phase 2+ Behavior

Button observes `UserCapabilityService.canAddCommitment` via Riverpod provider.

- **`true`** → button active, tap opens Commitment Form in create mode
- **`false`** → button dimmed, tap shows the blocked dialog

The button never evaluates any condition itself — it renders what the provider says and forwards taps to the form or the dialog.

---

## Blocked Dialog

Shown when the user taps the dimmed button. Content is determined by `UserCapabilityService.getBlockedReason()`. Always shows specific numbers — never vague.

### Performance threshold blocked

```
Not yet — but you're close.

Keep 50% of your commitments for 7 days
to spin a new one.

Your consistency: 43% → Goal: 50%
████████░░░░  Est. ~4 days

Slow down — it's a feature, not a bug. 🙂

[ Got it ]
```

### Rate limit blocked

```
Give it a few days.

You've added 3 commitments this week.
Adding more has a reverse effect —
weave before expanding.

Added this week: 3 of 3
Next slot available: Thursday

[ Got it ]
```

### Both blocked

Show the performance threshold dialog only — one problem at a time.

### Free tier ceiling reached (all gates green)

```
You've mastered your threads.

You're at the free tier limit (7 commitments).
Upgrade to Pro to keep expanding.

[ Upgrade to Pro ]   [ Not now ]
```

---

## Rules

- No gate logic inside this component — all decisions come from `UserCapabilityService`
- The current consistency score and rate limit values shown in the dialog are read from `UserCapabilityService.getBlockedReason()` return data — not fetched independently
- Phase 1: gate provider not wired, button always active

---

## Dependencies

- `UserCapabilityService.canAddCommitment` — Riverpod provider (Phase 2)
- `UserCapabilityService.getBlockedReason()` — dialog content (Phase 2)