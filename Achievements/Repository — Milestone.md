**File Name**: repository_milestone **Feature**: Achievements **Phase**: 2 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** storage for `MilestoneRecord`. Called only by `MilestoneService`.

---

## Operations

| Operation                                          | Input                     | Output                        |
| -------------------------------------------------- | ------------------------- | ----------------------------- |
| `saveMilestone(record)`                            | MilestoneRecord           | —                             |
| `milestoneExists(definitionId, streakCount)`       | definitionId, streakCount | bool — idempotency check      |
| `getMilestonesForCommitment(definitionId, limit?)` | definitionId              | List ordered by earnedAt desc |
| `getAllMilestones(limit?)`                         | optional limit            | List ordered by earnedAt desc |
| `deleteMilestonesForCommitment(definitionId)`      | definitionId              | —                             |

---

## Firestore Path

```
/users/{userId}/milestones/{id}
```

Covered by existing security rule.

---

## Rules

- Called only by `MilestoneService`
- Append-only except cascade delete on commitment permanent deletion
- `milestoneExists` always called before `saveMilestone`
- `deleteMilestonesForCommitment` called only on `InstancePermanentlyDeletedEvent`