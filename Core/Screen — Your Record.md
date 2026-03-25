**File Name**: screen_your_record **Feature**: core **Phase**: 1 (partial) · Phase 2 · Phase 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the user's complete progress picture in one scroll. A reflective screen — opened a few times a week, not daily. Shows commitment health, performance history, earned rewards, streak records, and progression journey across all commitments.

Not in bottom nav. Accessed via deep-link from notifications (cup earned, streak milestone) or from other screens that reference it. GoRouter deep-links: `/record` · `/record?section=cups` · `/record?section=streaks`.

---

## What It Will Contain

A scrollable screen assembled from multiple features. The exact layout and component details are not designed yet — this describes the intent for each section.

**Header** — a summary of the user's overall consistency and best streak across all commitments. Numbers here are the exception to the visuals-over-numbers rule — the user has explicitly navigated here to see their data.

**Commitment health** — the current state of each active commitment shown as a visual summary. Which commitments are on track, which need attention, how many slots are available.

**Performance history** — a calendar view of daily scores over time. Gives the user a visual sense of their consistency pattern — are they improving, declining, or steady?

**Streaks** — current and best streaks per commitment. Milestone achievements.

**Weekly cups** — a history of earned cups by week. The user's reward record over time.

**Progression teaser** — compact summary of the user's current progression level and points. Tap navigates to the full Progression screen. Phase 3 only.

---

## Expected Components

These are the components anticipated for this screen. None are fully designed yet — this list captures intent, not a final decision.

|Component|Purpose|Phase|
|---|---|---|
|Commitment Health|Colored dot per commitment showing current window performance|2|
|Performance Calendar (full mode)|Monthly calendar of daily scores, swipeable by month|2|
|Streak Summary (`component_streak_ui`)|Current and best streak per commitment|2|
|Weekly Cups|Cup history by week — link to Achievements screen for full list|2|
|Achievements link|Entry point to `screen_achievements` — full achievement history|2|
|Progression Teaser|Compact level summary, tap → Progression screen|3|

---

## What Is Not Designed Yet

The layout, component interactions, data sources, and section ordering are not defined here. This doc will be filled in progressively as each component is designed and the screen's shape becomes clear.