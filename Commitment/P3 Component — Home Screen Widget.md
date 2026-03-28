**File Name**: component_home_screen_widget **Feature**: Commitment **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** Android home screen widget that shows today's commitments and allows quick-logging without opening the app. Reduces logging friction to one tap from the home screen.

This is not an in-app screen — it is a native Android widget rendered on the device home screen. It is the lowest-friction entry point to logging, sitting outside the app entirely.

---

## What It Will Show

A compact view of today's active commitments with a quick-log button per row, and a summary of today's overall progress at the top. Tapping anywhere outside a quick-log button opens the app on the Dashboard.

---

## What Is Not Designed Yet

The exact layout, refresh mechanism, quick-log behavior, and data sources are not defined here. Those decisions depend on the Dashboard card interaction and the `defaultLogIncrement` field — neither of which are settled yet. This doc will be filled in when those are ready.

---

## Known Constraints

- Android only — no iOS equivalent planned
- Widget refresh is not real-time — periodic sync via WorkManager
- Available on all tiers — not a paid feature