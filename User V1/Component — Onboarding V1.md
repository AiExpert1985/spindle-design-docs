
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: User
**Phase**: 3
**Standalone:** yes — plugs in with zero changes to Phase 1 code. In Phase 1, users go directly to the dashboard.

**Purpose:** guides a new user to their first successful log in under 10 seconds. No explanation, no tutorial — the user understands the app by doing it, not by reading about it.


---

## Why Phase 2

Phase 1 is for internal testing with known users who can be explained to directly. Onboarding is designed for strangers installing the app cold — that's a Phase 2 concern when the app is ready for real users.

---

## When It Shows

On app first launch, if `UserProfile.hasCompletedOnboarding == false`. After completion, flag is set to true and never shown again.

In Phase 1 this flag is always null — onboarding never triggers. Zero behavior change.

---

## The Flow

### Step 1 — One question, one tap

```
What do you want to work on?

[ Exercise ]         [ Reading ]
[ Sleep ]            [ Nutrition ]
[ Avoid something ]  [ Custom ]
```

No form, no text input. One tap selects a category.

"Custom" opens a minimal name field — the only text input in the flow.

---

### Step 2 — Commitment created instantly

Each category maps to a smart default commitment definition:

|Category|Name|Type|Target|Recurrence|
|---|---|---|---|---|
|Exercise|Daily Walk|do|binary|daily|
|Reading|Read daily|do|binary|daily|
|Sleep|Sleep before midnight|do|binary|daily|
|Nutrition|Drink water|do|binary|daily|
|Avoid something|(name prompt)|avoid|0 (zero tolerance)|daily|
|Custom|(name prompt)|do|binary|daily|

Commitment created silently via `CommitmentService.createCommitment()`. User lands on dashboard immediately — no confirmation, no form.

---

### Step 3 — First log

User arrives on dashboard with one card:

```
TODAY  0%

Daily Walk     [ Done ✓ ]
```

User taps once. Score jumps. Log reward animation fires. First reward loop complete in under 10 seconds.

---

### Step 4 — Opt-in starters (after first log)

Shown once, immediately after the first successful log:

```
Want a few popular commitments to start with?

[ Read daily ]  [ Drink water ]  [ No alcohol ]  [ Skip ]
```

Opt-in only — never automatic. Each tap creates one commitment with smart defaults. "Skip" dismisses without creating anything.

After this step → `hasCompletedOnboarding: true` set on UserProfile.

---

## Unlock Engine Bypass

The first 3 commitments created during onboarding skip both unlock engine gates. Users must be able to set up initial commitments freely without earning the right first.

This bypass is enforced by `CommitmentService.createCommitment()` checking the onboarding flag — not by the unlock engine itself.

---

## Data Model Addition

One field added to `UserProfile` when this component is built:

```
hasCompletedOnboarding: bool?   // null in Phase 1, false on first launch, true after completion
```

No behavior change in Phase 1 — field is null and never checked.

---

## Service Logic

- App start → check `UserProfile.hasCompletedOnboarding`
- If false → show onboarding flow
- Category tap → `CommitmentService.createCommitment(smartDefault)`
- First log → show opt-in starters
- Opt-in tap → `CommitmentService.createCommitment(starterDefault)`
- After opt-in step → `UserService.completeOnboarding()` sets flag to true

---

## Dependencies

- `CommitmentService.createCommitment()` — creates commitments from defaults
- `UserService.completeOnboarding()` — sets `hasCompletedOnboarding: true`
- Dashboard screen — onboarding lands user here after Step 2