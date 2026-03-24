**File Name**: screen_dashboard **Feature**: core **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** the user's daily home base. The default screen when the app opens. Opened multiple times a day — primarily to log commitments, secondarily to check how today is going. Also the destination after day celebration.

The Dashboard is the logging surface. The Commitments screen is the management surface. These are separate screens with separate jobs — the Dashboard is never used to edit, freeze, or delete commitments, and the Commitments screen is never used to log.

---

## What It Will Contain

A view of today's commitments grouped by type (Do / Avoid), each displayed as an interactive card. The card is the primary surface for logging — the user swipe-logs directly from here without navigating anywhere else. This is the screen that owns the logging moment as described in `ux_principles` rule 1.

Above the cards, a summary of today's overall performance and recent weekly momentum — giving the user an at-a-glance sense of how today and this week are going before they start interacting with individual commitments.

---

## Entry Points

- App open — default screen
- After day celebration — celebration dismisses and lands here
- Bottom nav tab

---

## Expected Components

These are the components anticipated for this screen. None are fully designed yet — this list captures intent, not a final decision.

|Component|Purpose|
|---|---|
|Score Ring or equivalent|Today's overall performance across all commitments|
|Performance Calendar (compact)|Last 7 days of daily scores — weekly momentum at a glance|
|Commitment Card (interactive)|One card per active commitment — the logging surface|
|Log Reward Animation|Visual feedback after a positive log|
|Context Tags sheet|Tag sheet shown after certain log events|
|Streak milestone overlay|Shown when a streak milestone is reached on log|

---

## What Is Not Designed Yet

The specific layout, component interactions, data sources, and the exact logging interaction are not defined here. Those decisions depend on the logging components and the performance summary components — none of which are settled for Phase 1. This doc will be filled in when those components are ready.