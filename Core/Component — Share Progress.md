**File Name**: component_share_progress **Feature**: Core **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** lets users share a snapshot of their progress as an image via the native share sheet. Organic marketing — every share carries the Spindle identity. Free for all tiers.

Expected placement: share button on the Day Celebration screen (Template A) and Your Record screen (Template B).

---

## How It Works

A purpose-built card widget is rendered off-screen using `RepaintBoundary`, captured as an image, then the native share sheet is invoked. No backend required — purely local.

---

## Template A — Today Snapshot

```
Spindle · Today  83%

Morning Walk  ✓
Read 30 pages ✓
Coffee ≤1     ✓
No alcohol    ✓

"Your garment grew stronger today."
```

---

## Template B — Weekly Summary

```
Spindle · This week  76%

Mon ⬤  Tue ⬤  Wed ◉  Thu ·  Fri ●

21 days consistent.
```

---

## Share Button Locations

|Location|Template|Button label|
|---|---|---|
|Day Celebration screen|A|"Share today's result"|
|Your Record screen|B|"Share this week"|

---

## Rules

- Available to all tiers — not gated
- No backend writes — read and share only
- Identity statement at the bottom of every image is fixed — never removed

---

## Data Sources

|Data|Source|
|---|---|
|Today's score (Template A)|`PerformanceService.getDayScore(today)`|
|Today's commitment states (Template A)|`CommitmentIdentityService.getInstancesForDay(today)`|
|Week score (Template B)|`PerformanceService.getOverallWeekScore(weekStart)`|
|Week calendar dots (Template B)|`PerformanceService.getDayScore(date)` per day|

---

## Dependencies

- `PerformanceService` — scores for both templates
- `CommitmentIdentityService` — commitment states for Template A
- Native share sheet (`share_plus` package)