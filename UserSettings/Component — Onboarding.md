**File Name**: component_onboarding **Feature**: UserSettings **Phase**: 2 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** guides a new user to their first successful log in under 10 seconds. No explanation, no tutorial — the user understands the app by doing it, not by reading about it.

Expected placement: shown on first app launch when `UserSettingsProfile.hasCompletedOnboarding == false`. After completion, navigates to the Dashboard and never shows again.

---

## When It Shows

On first launch, if `UserSettingsProfile.hasCompletedOnboarding == false`. In Phase 1 this flag is always null — onboarding never triggers. Zero behavior change in Phase 1.

---

## The Flow

### Step 1 — One question, one tap

```
What do you want to work on?

[ Exercise ]         [ Reading ]
[ Sleep ]            [ Nutrition ]
[ Avoid something ]  [ Custom ]
```

No form, no text input. One tap selects a category. "Custom" opens a minimal name field — the only text input in the flow.

---

### Step 2 — Commitment created instantly

Each category maps to a smart default:

|Category|Name|Type|Target|Recurrence|
|---|---|---|---|---|
|Exercise|Daily Walk|do|binary|daily|
|Reading|Read daily|do|binary|daily|
|Sleep|Sleep before midnight|do|binary|daily|
|Nutrition|Drink water|do|binary|daily|
|Avoid something|(name prompt)|avoid|0 (zero tolerance)|daily|
|Custom|(name prompt)|do|binary|daily|

Commitment created via `CommitmentService.createCommitment()`. User lands on Dashboard immediately — no confirmation, no form.

---

### Step 3 — First log

User arrives on Dashboard with one card. Taps once. Log reward animation fires. First reward loop complete in under 10 seconds.

---

### Step 4 — Opt-in starters (after first log)

Shown once, immediately after the first successful log:

```
Want a few popular commitments to start with?

[ Read daily ]  [ Drink water ]  [ No alcohol ]  [ Skip ]
```

Opt-in only — never automatic. Each tap creates one commitment with smart defaults. "Skip" dismisses.

After this step → `UserSettingsService.completeOnboarding()` sets `hasCompletedOnboarding: true`.

---

## Onboarding Bypass

The first `AppConfig.onboardingBypassCount` commitments skip all `UserCapabilityService` gates. Bypass enforced by `UserCapabilityService` checking `hasCompletedOnboarding` before evaluating gates.

---

## Rules

- Never shown in Phase 1 — flag is null, never checked
- `CommitmentService.createCommitment()` is the only service called during commitment creation steps
- Opt-in starters are created identically to normal commitments — no special path

---

## Dependencies

- `CommitmentService.createCommitment()` — creates commitments from defaults
- `UserSettingsService.completeOnboarding()` — sets completion flag
- `UserSettingsProfile.hasCompletedOnboarding` — gate check on app start