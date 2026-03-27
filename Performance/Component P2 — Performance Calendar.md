**File Name**: component_performance_calendar **Feature**: Performance **Phase**: 2 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** shows daily performance as colored shapes for a given period. Gives the user an instant visual of consistency — am I trending up, how consistent have I been over time? One component, two display modes depending on placement.

---

## Two Display Modes

### Compact Mode

Fixed 7-day window. Bar heights represent score. No navigation.

```
Mon   Tue   Wed   Thu   Fri   Sat   Sun
 █     █     █     ░     █     —     —
```

- 7 columns, current day highlighted
- Bar height proportional to score
- Future days and days with no commitments: grey dash `—`
- Tap any bar → tooltip showing date and score
- Today's bar updates when `PerformanceUpdatedEvent` fires — provider rebuilds and re-fetches today's score

Expected placement: Dashboard, below the overall score summary.

---

### Full Mode

Monthly calendar grid, swipeable by month. Circle sizes represent score.

```
      ← November →
Mon   Tue   Wed   Thu   Fri   Sat   Sun
 ◉     ⬤    ◉     ·     ●     ⬤    ·
 ●     ◉    ·     ◉     ◉     ⬤    ●
```

- One circle per day, size proportional to score
- Neutral color throughout — size is the signal, not color
- No dot below threshold — silence, not punishment. See `ux_principles` rule 7.
- Swipe left / right to navigate months
- Tap any dot → tooltip showing date and score
- Fetches one month at a time — never preloads all history

Expected placement: Your Record screen, performance history section.

---

## Circle Sizes — Full Mode

|Score|Size|Meaning|
|---|---|---|
|< 40%|No dot|Silence|
|40–59%|Small ●|Some effort|
|60–79%|Medium ◉|Good day|
|≥ 80%|Large ⬤|Strong day|

---

## Bar Colors — Compact Mode

|Score|Color|
|---|---|
|< 50%|Red|
|50–79%|Amber|
|≥ 80%|Green|
|≥ 90%|Gold|

---

## Shared Behavior

- Tap any shape → tooltip showing date and score
- Today's entry updates when `PerformanceUpdatedEvent` fires — provider rebuilds, re-fetches today's score via `getDayScore(today)`
- Past days are static — fetched once, not streamed
- Frozen days shown as neutral — not counted against the user

---

## Data Sources

The component calls `getPerformanceForPeriod(from, to, definitionId?)` once per visible period. Per-day scores are derived from the returned instance data — not fetched with one call per day.

|Data|Source|
|---|---|
|Scores for a period|`PerformanceService.getPerformanceForPeriod(from, to, definitionId?)` — one-time read per period|
|Today's score live update|`PerformanceService.getDayScore(today)` — re-fetched on `PerformanceUpdatedEvent`|

The component accepts an optional `definitionId`. When provided, it shows only that commitment's daily scores — same visual, same interaction, different data scope.

---

## What Is Not Designed Yet

The exact provider structure and the integration with the Dashboard and Your Record screens are not defined here. Those decisions depend on both placements being designed. This doc will be filled in when those screens are ready.