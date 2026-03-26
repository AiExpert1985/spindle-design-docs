**File Name**: screen_commitment_form **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** create or edit a commitment. Flat form — all fields visible upfront. No staged reveal, no collapsing. For Phase 1 internal testing, a working form that produces a valid `CommitmentDefinition` is the only goal.

---

## Layout

```
┌─────────────────────────────────────────┐
│  New Commitment                    ✕    │
│                                         │
│  Name                                   │
│  [ What do you want to commit to? ]     │
│                                         │
│  Type                                   │
│  [ Do it ]        [ Stop it ]           │
│                                         │
│  Target value     Unit (optional)       │
│  [ 1           ]  [ pages, km, ...  ]   │
│                                         │
│  Recurrence                             │
│  [ Daily ▾ ]                            │
│                                         │
│  Window start     Window duration       │
│  [ 00:00       ]  [ 24 hours       ]    │
│                                         │
│  Warning notification                   │
│  [ On ]  [ Off ]                        │
│                                         │
│  Description (optional)                 │
│  [                                  ]   │
│                                         │
│              [ Create ]                 │
└─────────────────────────────────────────┘
```

---

## Fields

**Name** — required. What the user calls this commitment. Example: "Morning Run".

**Type** — required. Do it (build a habit) or Stop it (break one). Immutable after creation — selector is disabled in edit mode.

**Target value** — the number the user wants to reach (Do) or stay under (Avoid). Defaults to 1 (done / not done). Must be ≥ 0.

**Unit** — optional label for the target. Examples: pages, km, cups. No effect on calculations — display and future AI analysis only.

**Recurrence** — Daily, Weekly, or Specific days. Selecting Specific days reveals a day-picker row inline. Defaults to Daily.

**Window start** — when the activity window opens, in local time. Defaults from `UserCoreService.getActivityWindowDefaults(recurrence)`.

**Window duration** — how long the window lasts. Defaults from `UserCoreService.getActivityWindowDefaults(recurrence)`.

**Warning notification** — on or off. When on, a reminder fires at 3/4 of the window duration. Defaults from `UserCoreService.getProfile().warningNotificationsEnabled`. Maps to `activityWindow.warningEnabled`.

**Description** — optional. Personal note or motivation. Shown on the Commitment Detail screen. No effect on behaviour.

---

## Default Values

Defaults come from `UserDefaultPreferences` — never hardcoded. The values below reflect the global defaults.

|Field|Default|
|---|---|
|Target value|1|
|Unit|None|
|Recurrence|Daily|
|Window start|Midnight|
|Window duration|24 hours|
|Warning notification|On|
|Description|Empty|

---

## Edit Mode

Same form, pre-filled with the existing values. A notice appears at the top:

```
Changes apply from today forward.
Past history is preserved.
```

Type selector (Do / Stop it) is shown but disabled — immutable after creation. Save button label: "Save Changes".

---

## Navigation

- Opened from: Commitments screen Add button, Commitment Detail ⋮ menu Edit, long-press action menu on a commitment row
- On create / save → closes, returns to previous screen
- ✕ or back → closes without saving. Confirmation prompt if unsaved changes exist.

---

## Service Calls

- Create → `CommitmentService.createCommitment(definition)`
- Edit → `CommitmentService.updateCommitment(definition)`

Gate check happens before this screen opens (via the Add Commitment Button component). If blocked, the unlock dialog is shown and this screen never opens.

---

## Later Improvements

These are not in scope for Phase 1. Documented here so the intent is not lost.

**Collapsed advanced fields.** Window start, window duration, and warning notification are rarely changed. Hiding them behind a "More options" toggle reduces visual noise for most users. The form opens with just name, type, target, and recurrence visible.

**Smart name detection.** If the name contains a number and a unit ("Read 30 pages", "Walk 5km"), auto-fill the target value and unit fields and prompt the user to confirm. Reduces manual entry for the common case.

**Default log amount.** A convenience field: the pre-filled value used when the user swipe-logs from a commitment card. Does not affect the target. Useful once the swipe-log gesture is the primary logging path.

**Scheduled freeze day picker.** A day selector for auto-freezing the commitment on specific days each week. Example: auto-freeze on rest days. See `component_scheduled_freeze`.