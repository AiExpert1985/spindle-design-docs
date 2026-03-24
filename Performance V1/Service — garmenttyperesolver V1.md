**Created**: 18-Mar-2026 **Modified**: - **Feature**: Commitment **Phase**: 2 (rule-based) · Phase 3 (AI-assisted for Pro/Premium)

**Purpose:** decides which garment type is assigned to a commitment at creation time. The garment type is permanent — it becomes part of the commitment's visual identity and never changes, even if the commitment is edited.

---

## Abstraction

```
GarmentTypeResolver (abstract)
  → RuleBasedGarmentTypeResolver     (Phase 2 — deterministic rules)
  → AIGarmentTypeResolver            (Phase 3 — AI-assisted for Pro/Premium)
```

Phase 2 uses rules only. Phase 3 adds an AI implementation for paid users that can make more nuanced decisions based on the commitment name and context. The interface is the same — one function, one result.

---

## Garment Types

|Garment|Visual complexity|Assigned to|
|---|---|---|
|Thread bracelet|Simplest|Binary daily (done/not done, no unit)|
|Sock|Simple|Measurable daily, small target|
|Glove|Moderate|Measurable daily, larger target|
|Scarf|Moderate|Weekly commitment|
|Sweater|Complex|Long-running or high-target commitment|

**For avoid commitments:** same garment types apply, but the starting state is reversed — the garment begins fully complete (100%) and unravels toward empty.

---

## Core Function

### `resolveGarmentType(definition)`

Takes a `CommitmentDefinition` at creation time and returns a `GarmentType` enum value. Called once — the result is stored on the definition and never re-evaluated.

**Phase 2 — Rule-based logic:**

```
if measureUnit == null (binary):
  → thread_bracelet

if recurrenceType == weekly or windowSpansDays >= 7:
  → scarf

if target >= largeTargetThreshold (configurable, default: 10):
  → sweater

if target >= smallTargetThreshold (configurable, default: 3):
  → glove

else:
  → sock
```

Simple, deterministic, no network call. Every commitment gets a garment type instantly at creation.

**Phase 3 — AI-assisted (Pro/Premium only):**

Sends the commitment name, type, target, and unit to the AI. The AI returns a garment type with a one-line reason. The rule-based fallback applies if the AI call fails.

Example AI reasoning: _"Daily walk of 5km — a meaningful physical commitment requiring sustained effort. Sweater."_

The AI override is purely cosmetic — it does not affect any calculation. It makes the garment feel more personally chosen.

---

## Stored on CommitmentDefinition

```
garmentType: GarmentType    // assigned once at creation, never changes
```

`GarmentType` is an enum: `thread_bracelet | sock | glove | scarf | sweater`

---

## Rules

- Called only at commitment creation — never at edit, never at runtime display
- Result is stored permanently — the garment type is part of the commitment's identity
- Phase 3 AI call is optional enhancement — rule-based result is always the fallback
- Free tier always uses rule-based resolver regardless of phase
- The resolver has no knowledge of garment rendering — it returns a type, nothing more

---

## Dependencies

- `CommitmentService.createCommitment()` — calls this resolver before saving the definition
- AI API (Phase 3, Pro/Premium only) — for the AI-assisted implementation
- `UserService.canAccessFeature()` — determines which resolver implementation is active