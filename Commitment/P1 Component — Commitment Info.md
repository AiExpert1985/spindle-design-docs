**File Name**: component_commitment_info **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** displays the core facts about one commitment — what it is, how it is configured, and current window progress. Standalone — can be placed on any screen that needs to show commitment details.

Expected placement: Commitment Detail screen, below the app bar.

---

## UI

```
Morning Walk                          do · daily
Target: 5km  ·  Window: 6–10am
Today: 3 of 5km
```

Weekly commitment example:

```
Walk 30km                             do · weekly
Target: 30km
This week: 12 of 30km
```

- Name in large text
- Type badge and recurrence label on the same line
- Target, unit, and activity window on the next line (activity window omitted for weekly — not meaningful at daily granularity)
- Current window progress below — updates live as the user logs
- Progress label is "Today" for daily and specific-day commitments, "This week" for weekly

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
|Window progress|`PerformanceService.getWindowProgress(definitionId)` — stream via provider|

Progress is rendered as `"${periodLabel}: ${logged} of ${target}${unit ?? ''}"`. The component does no calculation — `WindowProgress` supplies all values ready to display. Null result (no pending instance) renders nothing in the progress line.

---

## Rules

- Receives its data from the parent screen's provider — does not fetch independently
- Window progress updates live — driven by `PerformanceUpdatedEvent` via Riverpod provider
- Description section is hidden entirely if `description` is null
- Progress line hidden entirely if `getWindowProgress()` returns null

---

## Later Improvements

**Streak display.** Current streak and personal best below the progress line. Requires the Streak feature (Phase 2).