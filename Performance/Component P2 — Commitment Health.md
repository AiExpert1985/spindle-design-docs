**File Name**: component_commitment_health **Feature**: Performance **Phase**: 2 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** shows the current performance state of each active commitment as a colored dot. Gives the user an instant snapshot of which commitments are on track and which need attention.

Expected placement: Your Record screen, in the commitment health section.

---

## UI

One colored dot per active commitment.

```
🟢  ⭐  🟠  🟢  🟢
5 active commitments
```

---

## Dot Colors

|Score|Color|Meaning|
|---|---|---|
|< 40%|🟠 Orange|Needs attention|
|40–79%|🟢 Green|On track|
|≥ 80%|⭐ Gold|Exceeding target|

Each dot reflects `livePerformance` from the commitment's current pending instance — the same value that drives the progress bar and score ring elsewhere. No separate calculation.

---

## Rules

- Active commitments only — frozen and completed are not shown
- Updates live while the screen is open — driven by `PerformanceUpdatedEvent` via Riverpod
- Read-only — no gestures in Phase 2

---

## Data Sources

|Data|Source|
|---|---|
|Active commitments|`CommitmentService.watchActiveCommitments()` — stream|
|Current instance per commitment|`CommitmentIdentityService.watchInstancesForDay(today)` — stream, read `livePerformance`|

---

## Later Improvements

**Tap to detail.** Tap any dot → navigates to Commitment Detail screen for that commitment.

**Slot count.** Empty circles for available unused slots alongside the filled dots. Requires `UserCapabilityService` for the active count and tier ceiling. Only meaningful for free tier users approaching their limit.