**File Name**: ux_principles **Phase**: All phases **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** non-negotiable implementation rules. Read before writing any UI code. These principles ensure a consistent, fast, and honest user experience across all screens and phases.

---

## 1. The Logging Moment Is Sacred

Every successful log must execute in this exact order:

1. `HapticFeedback.lightImpact()` — instant, before any animation
2. Floating value animation — "+5 pages" rises ~80px, fades 600ms
3. Progress bar — fills smoothly 400ms, never jumps
4. Score ring — animates 400ms, clockwise, color transitions at thresholds
5. Inline threshold message — appears on milestone crossings, fades 2s

Never show "Logged successfully." Always show what changed: `"15 → 20 pages · 64% → 70%"`

---

## 2. Positive Only — Silence on Negative

|Moment|Response|
|---|---|
|Positive log|Floating value + ring animates|
|Streak milestone|Full overlay with badge|
|Day ends ≥ 60%|Day celebration screen|
|Miss / swipe left|Silent → recovery sheet|
|Limit exceeded|Silent → card turns amber|
|"I slipped"|Silent → context tag sheet|
|Day ends < 60%|Complete silence|

No red flashes. No shake animations. No "You failed."

---

## 3. Logging Takes ≤ 5 Seconds

Acceptable paths: swipe right (1 tap) · widget (1 tap) · notification button (1 tap) · long press → confirm (3 taps).

Unacceptable: requiring navigation to the detail screen to log.

---

## 4. Action Before Configuration

Onboarding: log before seeing any form. Commitment form: Stage 1 is two decisions. Advanced options: contextual only, never upfront.

---

## 5. Animation Performance

- All animations ≤ 600ms
- Never block user input during animation
- Drop frames → skip animation entirely. Janky is worse than missing.
- Use `flutter_animate` throughout
- Test on low-end Android before shipping

---

## 6. Analytical, Never Moral

Every message describes what happened. It never evaluates the user.

"Commitment missed" not "You failed." "What pattern caused this?" not "Why did you fail?"

---

## 7. Silence Is a Valid Response

Below-threshold days get no animation, no message, no color. Silence is non-judgmental. The app does not shame users for bad days.

---

## 8. Show What Changed, Not That Something Happened

Never: "Saved." "Logged successfully." "Done." Always: what actually changed — scores, values, progress.

---

## 9. Your Record Is Not in Bottom Nav

Accessed via the Commitments screen teaser or notifications. GoRouter deep-link: `/record`, `/record?section=cups`, `/record?section=streaks`.

---

## 10. Visuals Over Numbers

The system calculates accurate scores internally, but users experience results through visuals — garment fill, circular arc, cup level, streak count. Raw percentages are almost never the right thing to show a user.

When a number must be shown, it should be a meaningful real-world value (pages read, km run, streak days) — not a calculated score percentage.

**Rule:** before displaying any calculated number to the user, ask whether a visual representation would communicate it better. In most cases it will. Score percentages drive garment state, ring fill, and cup thresholds — the user sees the result, not the number behind it.

The one exception: when the user explicitly asks to see their performance data (Your Record screen, commitment detail stats). Even then, prefer trend indicators (improving / steady / declining) over raw percentages where possible.