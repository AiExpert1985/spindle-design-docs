**File Name**: component_commitment_info **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** displays the core facts about one commitment — what it is, how it is configured, and today's progress. Standalone — can be placed on any screen that needs to show commitment details.

Expected placement: Commitment Detail screen, below the app bar.

---

## UI

```
Morning Walk                          do · daily
Target: 5km  ·  Window: 6–10am
Today: 3 of 5km
```

- Name in large text
- Type badge and recurrence label on the same line
- Target, unit, and activity window on the next line
- Today's progress below — updates live as the user logs

Description is shown below if set:

```
"Run before work — non-negotiable."
```

---

## Fields

|Field|Source|
|---|---|
|Name|`CommitmentDefinition.name`|
|Type|`CommitmentDefinition.commitmentType`|
|Recurrence|`CommitmentDefinition.recurrence`|
|Target + unit|`CommitmentDefinition.target.value` + `target.measureUnit`|
|Window|`CommitmentDefinition.activityWindow.startMinutes` + `activityWindow.durationMinutes`|
|Description|`CommitmentDefinition.description`|
|Today's progress|`CommitmentInstance.livePerformance` — stream|

---

## Rules

- Receives its data from the parent screen's provider — does not fetch independently
- Today's progress updates live — driven by the instance stream already open on the parent screen
- Description section is hidden entirely if `description` is null

---

## Later Improvements

**Streak display.** Current streak and personal best below the progress line. Requires the Rewards feature — streak data lives there, not on the instance or definition.