**File Name**: service_garment_type_resolver **Feature**: Garment **Phase**: 3 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** assigns a garment type to a commitment at creation time. Called once — the result is stored on `GarmentProfile` and never changes. The garment type is part of the commitment's permanent visual identity.

Abstract interface — the resolution logic can be replaced without touching `GarmentService` or any display component.

---

## Abstraction

```
GarmentTypeResolver (abstract)
  → RuleBasedGarmentTypeResolver     // deterministic, synchronous
  → AIGarmentTypeResolver            // AI-assisted, Pro/Premium only (later)
```

---

## Interface

```dart
abstract class GarmentTypeResolver {
  GarmentType resolve(CommitmentDefinition definition);
}
```

Synchronous. The rule-based implementation returns instantly with no external calls. The AI implementation (later) uses the rule-based result as its synchronous fallback — the AI call happens asynchronously but the resolver always returns a valid result immediately.

---

## Garment Types

|Type|Visual|Assigned to|
|---|---|---|
|`threadBracelet`|Simplest — circular wrist band|Binary daily (no unit)|
|`sock`|Simple silhouette|Measurable daily, small target|
|`glove`|Moderate silhouette|Measurable daily, larger target|
|`scarf`|Long rectangular wrap|Weekly commitment|
|`sweater`|Complex silhouette|Long-running or high-target commitment|

For Avoid commitments: same types apply, starting state reversed — garment begins complete and unravels.

---

## Rule-Based Implementation

```
if target.measureUnit == null (binary):
  → threadBracelet

if recurrence == Weekly:
  → scarf

if target.value >= largeTargetThreshold (AppConfig, default: 10):
  → sweater

if target.value >= smallTargetThreshold (AppConfig, default: 3):
  → glove

else:
  → sock
```

Simple and deterministic. Every commitment gets a type instantly at creation with no network call.

---

## AI-Assisted Implementation (Later)

Sends commitment name, type, target, and unit to the AI. Returns a `GarmentType` with a one-line reason. Rule-based result is always the synchronous fallback if the AI call fails or is unavailable.

The AI assignment is purely cosmetic — it makes the garment feel personally chosen. It does not affect any calculation.

Example AI reasoning: "Daily walk of 5km — a meaningful physical commitment requiring sustained effort. Sweater."

Available to Pro and Premium users only. Free tier always uses rule-based resolver.

---

## Rules

- Called only once — at garment creation via `GarmentService._onInstanceCreated()`
- Result stored on `GarmentProfile.garmentType` — never re-evaluated
- Synchronous interface — AI implementation uses rule-based result as immediate fallback
- The resolver has no knowledge of rendering — returns a type, nothing more

---

## Dependencies

- `CommitmentDefinition` — input only, read at creation time
- AI API — AI implementation only, Pro/Premium
- `UserCoreService.getTier()` — determines which implementation is active
- `AppConfig` — threshold constants