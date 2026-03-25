**File Name**: model_milestone_record **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** a permanent record of one streak milestone reached for a specific commitment. Written once when the streak crosses a milestone threshold.

---

## Fields

```
MilestoneRecord
  id: String
  definitionId: String
  commitmentName: String   // snapshotted at write time — survives commitment rename
  streakCount: int         // the streak value that triggered this milestone
  earnedAt: DateTime
  createdAt: DateTime
```

- **commitmentName** — snapshotted at write time so the record remains readable even if the commitment is later renamed or deleted.
- **streakCount** — the exact streak value at the milestone moment. Matches one of the values in `AppConfig.streakMilestones`.
- **earnedAt** — when the milestone was reached — the moment the window closed with the qualifying streak count.

---

## Milestone Thresholds

Defined in `AppConfig.streakMilestones`. Default values: `[3, 5, 7, 10, 14]`.

---

## Rules

- One record per milestone per commitment — `MilestoneService` checks idempotency before writing
- Append-only — never edited after creation
- Written only by `MilestoneService`
- Deleted when `InstancePermanentlyDeletedEvent` fires for this `definitionId`