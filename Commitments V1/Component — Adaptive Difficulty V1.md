
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 2

**Standalone:** yes — shown conditionally on commitment detail screen, no changes to other components.

**Purpose:** suggests raising or lowering a commitment's target based on recent performance. Prevents the app from feeling stale when a commitment becomes too easy or too hard. Tapping a suggestion pre-fills the edit form.



---

## When It Appears

Shown on the commitment detail screen under the performance summary:

**Too easy** — exceeded target 7 or more of the last 10 instances:

```
You've hit this 9/10 times. Ready to raise it?
[ 6km ]  [ 7km ]  [ Keep as is ]
```

**Too hard** — missed 5 or more of the last 7 instances:

```
This has been tough lately. Want to adjust?
[ 3km ]  [ 2km ]  [ Keep trying ]
```

Never shown if fewer than 10 instances exist — not enough data.

---

## Suggested Values

Suggestions are calculated from the current target:

- Raise: +20% and +40% of current target, rounded sensibly
- Lower: -20% and -40% of current target, rounded sensibly

---

## Tap Behavior

Tapping any value → opens Commitment Form in edit mode with the suggested value pre-filled. User can accept, adjust, or cancel. Nothing changes until the form is saved.

---

## Service Logic

- Fetch last 10 instances → `CommitmentService.getInstancesForPeriod(last 10)`
- Evaluate conditions locally — no new service function needed
- On tap → navigate to Commitment Form with pre-filled target

---

## Dependencies

- `CommitmentService.getInstancesForPeriod()` — recent instance data
- Commitment Form screen — navigation target on tap