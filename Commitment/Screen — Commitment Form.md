**File Name**: screen_commitment_form **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 23-Mar-2026

---

**Purpose:** create or edit a commitment. Designed for maximum simplicity — a new commitment requires exactly two decisions. Everything else defaults silently. Power users can expand to configure more.

---

## Layout — Create Mode (Stage 1)

```
┌─────────────────────────────────────────┐
│  New Commitment                    ✕    │
│                                         │
│  [ What do you want to commit to? ]     │
│                                         │
│  [ Do it ]        [ Stop it ]           │
│                                         │
│  ▼ More options                         │
│                                         │
│              [ Create ]                 │
└─────────────────────────────────────────┘
```

Two decisions — name and type. Everything else defaults silently. Tap Create → commitment saved, screen closes, user returns to previous screen with the new commitment visible.

---

## Stage 2 — Smart Detection (conditional)

If the name contains a number and a unit ("Walk 5km", "Read 30 pages"), a prompt appears inline below the name field:

```
Track progress toward 5km?
[ Yes — track amount ]   [ No — just done / not done ]
```

If the user chooses Yes, target value and unit fields appear pre-filled from the parsed name. This is a convenience shortcut — if parsing is wrong or ambiguous, the fields are editable.

---

## Stage 3 — More Options (collapsed by default)

Tapping "▼ More options" expands:

```
Window start time      [ 6:00 AM ]
Window duration        [ 4 hours ]
Recurrence             [ Daily ▾ ]
Warning notification   [ On  ▾  ]
Default log amount     [ 1 ]
Description            [ optional note ]
```

90% of users never open this. Defaults are sensible for most commitments.

**Warning notification** is a toggle (on/off) that maps to `activityWindow.warningEnabled`. When on, a reminder notification fires at 3/4 of the window duration. The global default comes from `UserDefaultPreferences.warningNotificationsEnabled`. The user can override it here per commitment.

**Default log amount** is the pre-filled value used when the user swipe-logs from the commitment card. Does not affect target — it is a UI convenience value only. Defaults to 1.

**Recurrence** options: Daily · Weekly · Specific days. Selecting Specific days reveals a day-picker row.

**Description** is optional. Shown on the Commitment Detail screen. No effect on behaviour.

---

## Default Values

Defaults come from `UserDefaultPreferences` — never hardcoded. The values below reflect the global defaults.

|Field|Default|
|---|---|
|Target value|1 (done / not done)|
|Unit|None|
|Window start|Midnight (full day)|
|Window duration|24 hours|
|Recurrence|Daily|
|Warning notification|On|
|Default log amount|1|

---

## Edit Mode

Same form, pre-filled with the existing commitment's values. A notice appears at the top:

```
Changes apply from today forward.
Past history is preserved.
```

`commitmentType` (Do / Stop it) is shown but disabled — it is immutable after creation. Save button label: "Save Changes".

---

## Scheduled Freeze Days (Phase 2)

A day-picker appears inside More Options in Phase 2:

```
Auto-freeze on:
[ Mon ] [ Tue ] [ Wed ] [ Thu ] [ Fri ] [ Sat ] [ Sun ]
```

See `scheduled_freeze` component.

---

## Navigation

- Opened from: Commitments screen Add button, Commitment Detail ⋮ menu Edit option, long-press action menu on a commitment row
- On create / save → closes, returns to previous screen
- ✕ or back → closes without saving. If unsaved changes exist, a confirmation prompt appears before dismissing.

---

## Service Calls

- Create → `CommitmentService.createCommitment(definition)`
- Edit → `CommitmentService.updateCommitment(definition)`

In create mode, the Add button gate is checked before this screen opens. If any gate is blocked, the unlock dialog is shown instead of opening the form. See `unlock_engine` component.