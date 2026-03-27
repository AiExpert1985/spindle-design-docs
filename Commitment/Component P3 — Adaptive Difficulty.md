**File Name**: component_adaptive_difficulty **Feature**: Commitment **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** surfaces an observation when a commitment's target appears consistently too easy or too hard. Advice only — no navigation, no pre-filling. The user reads it and decides what to do.

Expected placement: commitment detail screen, below the commitment facts. Shown conditionally — only when the condition is met and the user has not dismissed it.

---

## When It Appears

Requires at least 10 closed instances — not enough data before that.

**Too easy** — target exceeded in 7 or more of the last 10 instances:

```
You've been hitting this easily — 9 out of 10 times.
You might be ready for a higher target.
                                        [ Dismiss ]
```

**Too hard** — target missed in 5 or more of the last 7 instances:

```
This one has been consistently tough lately — missed 5 of the last 7.
A lower target might be more sustainable.
                                        [ Dismiss ]
```

Only one card shown at a time. If both conditions are somehow met, show too-hard only — adjusting down is more urgent than adjusting up.

---

## Dismiss Behavior

Tapping Dismiss hides the card permanently for this commitment. It never reappears for the same commitment unless the user edits the target — a target change resets the dismissed state, since the observation no longer applies.

---

## Rules

- Advice only — no navigation, no form pre-filling
- Never shown for frozen or completed commitments
- Dismissed state stored per commitment — not a global setting
- Requires 10 closed instances minimum before evaluating
- Evaluation is read-only — writes nothing except the dismissed flag

---

## Data Sources

|Data|Source|
|---|---|
|Last 10 closed instances|`CommitmentIdentityService.getInstances(from, to, definitionId)`|
|Dismissed state|Stored on `CommitmentDefinition` or local preference — decided at implementation|