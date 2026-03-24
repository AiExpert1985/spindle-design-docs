
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: User
**Phase**: 3
**Standalone:** yes — tier checks happen inside each gated feature, not in a central place.

**Purpose:** lets users share a snapshot of their progress as an image via the native share sheet. Organic marketing — every share carries the Spindle identity. Free for all tiers.

---

## How It Works

A purpose-built card widget is captured as an image off-screen, then the native share sheet is invoked. The user sees WhatsApp, Telegram, Messenger, and any other app they have installed. No backend required.

---

## Template A — Today Snapshot

```
Spindle · Today  83% ⭐

Morning Walk  ✓
Read 30 pages ✓
Coffee ≤1     ✓
No alcohol    ✓

"Your rope grew stronger today."
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

| Location | Template | Button label |
|---|---|---|
| Encouragement screen | A | "Share today's result" |
| Your Record | B | "Share this week" |
| Weekly report screen | B | "Share my review" |

---

## Identity Statement

The phrase at the bottom of every share image is intentional — it reinforces the Spindle metaphor every time it's shared outside the app.

---

## Service Logic

- Render the appropriate card widget off-screen using `RepaintBoundary`
- Capture as image
- Invoke native share sheet with the image

---

## Dependencies

- `CommitmentService.getDayScore` — for Template A today score
- `CommitmentService.getPeriodAverageScore` — for Template B week score
- Native share sheet (`share_plus` package)
- No repository writes — purely read and share
