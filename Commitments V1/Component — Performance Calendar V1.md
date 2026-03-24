
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: Commitment
**Phase:** 1 (compact mode) · Phase 2 (full mode)

**Standalone:** yes — can be placed on any screen that needs a performance history visual.


**Purpose:** shows daily performance as colored shapes for a given period. Gives the user an instant visual of consistency — am I trending up, and how consistent have I been over time? One component, two display modes depending on where it is placed.

---

## Why One Component

Both the dashboard weekly view and the Your Record monthly calendar show the same thing — daily scores as colored shapes. Keeping them as one component means one place to update color thresholds, one place to update tooltips, one data source.

---

## Two Display Modes

### Compact Mode — Dashboard (Phase 1)

Fixed 7-day window. Bar heights represent score. No navigation.

```
Mon   Tue   Wed   Thu   Fri   Sat   Sun
 █     █     █     ░     █     —     —
72%   65%   81%   54%   63%
```

- 7 columns, current day highlighted
- Bar height proportional to score
- Future days / no commitments: grey dash `—`
- Tap any bar → tooltip: `"Tuesday · 65%"`
- Today's bar updates live as user logs

---

### Full Mode — Your Record (Phase 2)

Monthly calendar grid, swipeable by month. Circle sizes represent score.

```
      ← November →
Mon   Tue   Wed   Thu   Fri   Sat   Sun
 ◉     ⬤    ◉     ·     ●     ⬤    ·
 ●     ◉    ·     ◉     ◉     ⬤    ●
```

- One circle per day, size proportional to score
- Neutral blue-grey color throughout — size is the signal, not color
- No dot below threshold — silence, not punishment
- Swipe left/right to navigate months
- Tap any dot → tooltip: `"Nov 28 · 84%"`
- Fetches one month at a time — never preloads all history

---

## Circle Sizes (Full Mode)

|Size|Score|Meaning|
|---|---|---|
|No dot|< 40%|Silence|
|Small ●|40–59%|Some effort|
|Medium ◉|60–79%|Good day|
|Large ⬤|≥ 80%|Strong day|

---

## Bar Colors (Compact Mode)

|Score|Color|
|---|---|
|< 50%|Red|
|50–79%|Amber|
|≥ 80%|Green|
|≥ 90%|Gold|

---

## Shared Behavior

- Tap any shape → tooltip showing date and score
- Today's entry updates live via Riverpod
- Past days are static — fetched once, not streamed
- Frozen days shown as neutral — not counted against the user

---

## Service Logic

**Compact mode:**

- Fetches last 7 day scores → `CommitmentService.getWeeklyScores()`
- Today's score → `CommitmentService.getDayScore(today)` via stream

**Full mode:**

- Fetches one month of scores → `CommitmentService.getMonthScores(month, year)`
- Fetches new month on navigation — never fetches all history upfront

---

## New Function Needed in CommitmentService

### `getMonthScores(month, year)`

Returns daily scores for every day in the given month. Used by full mode when user navigates to a month. Returns a map of date → score. Days with no active commitments return null.

---

## Dependencies

- `CommitmentService.getWeeklyScores()` — compact mode
- `CommitmentService.getMonthScores(month, year)` — full mode
- `CommitmentService.getDayScore(today)` — live today update
- Presentation layer — Riverpod watches today's score for live update

---

## Per-Commitment Mode

When placed on a commitment detail screen, the calendar receives a `definitionId` and shows only that commitment's daily scores — not overall day scores.

Uses `CommitmentService.getCommitmentMonthScores(id, month, year)` instead of `getMonthScores()`. Same visual, same interaction, different data scope.