
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 1

**Standalone:** yes — can be added to any screen that needs to display commitment details.

**Purpose:** displays the core facts about one commitment — what it is, how it's configured, and its current streak. The first thing a user sees when opening a commitment detail screen.


---

## UI

```
Morning Walk                          do · daily
Target: 5km  ·  Window: 6–10am
3 of 5km today

Streak: 7 days 🥇   Best: 12 days
```

- Commitment name in large text
- Type badge (do / avoid) and recurrence (daily / weekly / custom)
- Target and window shown concisely
- Current progress for today's instance
- Current streak and personal best below

---

## What It Shows

|Field|Source|
|---|---|
|Name|`CommitmentDefinition.name`|
|Type|`CommitmentDefinition.commitmentType`|
|Recurrence|`CommitmentDefinition.recurrenceType`|
|Target + unit|`CommitmentDefinition.target` + `measureUnit`|
|Window|`CommitmentDefinition.windowStartMinutes` + `windowDurationHours`|
|Today's progress|Current `CommitmentInstance.progressPercent`|
|Current streak|`CommitmentDefinition.currentStreak`|
|Best streak|`CommitmentDefinition.bestStreak`|

---

## Service Logic

- Fetch definition → `CommitmentService.getActiveCommitments()` filtered by id
- Fetch today's instance → `CommitmentService.getTodayInstances()` filtered by definitionId

---

## Dependencies

- `CommitmentService` — definition and today's instance
- Presentation layer — Riverpod watches today's instance for live progress update