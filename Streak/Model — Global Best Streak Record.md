**File Name**: model_global_best_streak_record **Feature**: Streak **Phase**: 2 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the single best streak ever achieved across all commitments for this user. One record per user. Updated whenever any commitment sets a new personal best. Used by the Your Record screen.

---

## Fields

```
GlobalBestStreakRecord
  definitionId: String       // which commitment holds the all-time best
  commitmentName: String     // snapshotted at write time — readable after rename or deletion
  streakDays: int            // the best streak value
  achievedAt: DateTime       // when this best was reached
  createdAt: DateTime
  updatedAt: DateTime
```

- **definitionId** — the commitment that achieved this best. Updated whenever a new best is set by any commitment.
- **commitmentName** — snapshotted at write time so the record remains readable after the commitment is renamed or deleted.
- **streakDays** — the all-time best streak count across all commitments.
- **achievedAt** — when the best was reached. Used for display ("personal best set on March 3rd").
- **createdAt** — set on first ever personal best. Immutable.
- **updatedAt** — updated whenever a new best is set.

---

## Rules

- One record per user — created on the first ever streak milestone, updated when a new best is set
- Written only by `StreakService`
- Never deleted — represents the user's lifetime achievement
- `commitmentName` snapshotted at write time — not updated if the commitment is later renamed