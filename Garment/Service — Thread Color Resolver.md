**File Name**: service_thread_color_resolver **Feature**: Garment **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** assigns thread colors to a commitment's garment at creation time. Called once — the colors are stored on `GarmentProfile` and never change. Thread colors are the commitment's visual fingerprint — consistent across all representations of the same commitment.

Abstract interface — the color assignment strategy can be replaced without touching `GarmentService` or any renderer.

---

## Abstraction

```
ThreadColorResolver (abstract)
  → RandomVividColorResolver       // curated random palette from vivid color set
  → UserChosenColorResolver        // user picks colors from a palette (later)
  → AIColorResolver                // AI suggests colors based on commitment name (later)
```

---

## Interface

```
resolve(definitionId: String) → List<String>
```

Returns a list of hex color strings. The number of colors returned matches the maximum thread count for the assigned garment type — enough colors to fill the garment completely.

---

## Random Vivid Implementation

Selects colors randomly from a curated set of vivid, distinct hues. The set is hand-picked to ensure colors look good together and are visually distinguishable on the garment canvas.

Rules for the random selection:
- No two adjacent colors in the list are too similar in hue
- Colors are saturated and warm — consistent with the handcraft aesthetic
- The same `definitionId` always produces the same colors — uses the ID as a seed for deterministic randomness

Deterministic seeding means the same commitment always gets the same colors, even if the garment is recreated after a reinstall or migration.

---

## Rules

- Called only once — at garment creation
- Result stored on `GarmentProfile.threadColors` — never re-evaluated
- Colors are immutable after assignment — they are part of the commitment's visual identity
- The resolver has no knowledge of rendering — returns hex strings, nothing more

---

## Dependencies

- None for the random implementation
- `UserService.getTier()` — for future implementations that require tier checks
