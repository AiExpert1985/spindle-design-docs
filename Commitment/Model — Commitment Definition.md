**File Name**: model_commitment_definition **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the template for a commitment. Defines what a commitment is, what the user wants to achieve, when it applies, and its current state. No activity or performance data lives here — those belong to other features.

---

## Two Vocabularies, One Feature

The definition is what the user configured — their commitment as they understand it. The instance is what actually happened on a given day. These serve different audiences:

- **UI reads the definition** — to show the user what they committed to: name, type, target, recurrence, window. The user configured these, and the UI reflects that configuration.
- **Upper features work with instances** — Performance, Notifications, Streaks, Garment all react to what happened, not what was configured. They read instances, never the definition.

This separation means the definition never changes retroactively — past history is always measured against the target that was in effect at the time, which lives on the instance.

---

## Commitment Type

A commitment is either something the user wants to do or something they want to avoid.

**Do** — a habit or goal the user wants to build and maintain. Example: read every day, run 5km, drink 8 glasses of water.

**Avoid** — a habit the user wants to eliminate or limit. Example: stop smoking, limit coffee to one cup, reduce screen time.

`commitmentType` is immutable after creation — changing it would make all historical performance meaningless.

---

## Recurrence

How often the commitment repeats. Sealed class because `SpecificWeekDays` needs to carry the list of days while `Daily` and `Weekly` do not. See `architecture_rules` §11.

```
sealed class Recurrence
  Daily
  Weekly
  SpecificWeekDays(weekDays: List<DayOfWeek>)
```

**Daily** — the user wants to act every day. Example: read 20 pages, no smoking today.

**Weekly** — the user has a weekly target to reach across the whole week. Example: walk 30km this week. The user decides when to act.

**SpecificWeekDays** — the user wants to act only on chosen days. Example: go to the gym on Sunday, Tuesday, and Friday.

`SpecificWeekDays` is named to allow future addition of `SpecificMonthDays` without a model change.

Recurrence is mutable. Changes apply from today forward — past records are not affected.

---

## Target

The numerical value the user wants to reach (Do) or stay under (Avoid).

```
Target
  value: double
  measureUnit: String?
```

- **value** — what the user is measuring. Always >= 0. Examples: 20 (pages), 5.0 (km), 1 (done/not done), 0 (avoid entirely).
- **measureUnit** — optional label. Examples: pages, km, cups. No effect on calculations — used for display and future AI analysis.

For Do: success means logging at or above the target. For Avoid: success means staying at or below. A target of 0 on Avoid means any logged amount is a breach.

Target is editable. Changes apply from today forward.

---

## Activity Window

The time of day the user prefers to work on this commitment, and the notification settings for that window.

```
ActivityWindow
  startMinutes: int
  durationMinutes: int
  warningEnabled: bool
```

- **startMinutes** — when the window opens, in minutes from midnight. Example: 360 = 06:00.
- **durationMinutes** — how long the window lasts. Example: 240 = 4 hours (06:00–10:00).
- **warningEnabled** — whether to send a reminder notification at 3/4 of the window duration. Lives here because it is a property of this window's notification behaviour — it has no meaning outside it.

Default values come from user settings. The user can override per commitment in advanced settings.

The activity window affects only notification timing — no effect on scoring or target calculation.

---

## Commitment State

The current lifecycle state of the commitment. Always exactly one state.

```
enum CommitmentState { active, frozen, completed, deleted }
```

**active** — running normally. The user receives notifications and is expected to log activity.

**frozen** — temporarily paused. No new instances generated. No notifications sent. Resumable at any time.

**completed** — deliberately finished. Treated like frozen operationally, but shown as an achievement in the user's record.

**deleted** — soft delete. Hidden from normal views. All closed instance history is preserved and the commitment is restorable from the Recycle Bin. The pending instance is cleared on soft delete — it has no successor until the commitment is restored. Permanent deletion is a separate irreversible action that removes all data.

State changes are recorded in the transition log.

---

## State Transition Log

Every state change is recorded. Gives a complete history of when a commitment was active, paused, or finished.

```
StateTransitionRecord
  id: String
  definitionId: String
  fromState: CommitmentState?
  toState: CommitmentState
  timestamp: DateTime
  reason: String?
```

- **fromState** — the previous state. Null on the first record (creation).
- **reason** — optional user note. Example: "taking a break while travelling."

Append-only. Used by the calendar to explain gaps and by Your Record for the lifecycle timeline.

---

## Fields

```
CommitmentDefinition
  id: String
  name: String
  description: String?
  commitmentType: CommitmentType
  recurrence: Recurrence
  target: Target
  activityWindow: ActivityWindow
  commitmentState: CommitmentState
  createdAt: DateTime
  updatedAt: DateTime
```

- **id** — unique identifier. Immutable.
- **name** — what the user calls this commitment. Required. Example: "Morning Run".
- **description** — optional. User adds context or personal motivation. Displayed on the detail screen only — no effect on any calculation.
- **commitmentType** — do or avoid. Immutable after creation.
- **recurrence** — sealed class. Daily, Weekly, or SpecificWeekDays. Mutable.
- **target** — typed class. The measurement goal. Editable — changes apply from today forward.
- **activityWindow** — typed class. Preferred time of day and notification settings. Editable. Affects notifications only.
- **commitmentState** — current lifecycle state. Only changed through `CommitmentService` functions.
- **createdAt** — immutable.
- **updatedAt** — last modified timestamp.

---

## Rules

- `CommitmentService` is the only writer of this model
- `commitmentType` is immutable after creation
- Target and recurrence changes apply from today forward — past records never modified
- State transitions only through `CommitmentService` — never by writing `commitmentState` directly
- UI reads the definition to show the user what they configured
- Upper features (Performance, Notifications, Streaks, Garment) work with instances — never the definition
- No external feature writes to this model