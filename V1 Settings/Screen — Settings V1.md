**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Settings **Phase**: 1 (temporal) · Phase 2 (notifications, preferences) · Phase 3 (account, subscription, storage)

**Purpose:** lets users configure app behavior and manage their account. Accessed from bottom nav tab 4.

---

## Screen Layout

```
┌─────────────────────────────────────────┐
│  SETTINGS                               │
│                                         │
│  CALENDAR & TIME             Phase 1    │
│  Week starts on              Monday  >  │
│  Rest days              Sat, Sun  >     │
│  Day resets at            Midnight  >   │
│  Waking hours           7am – 10pm  >  │
│                                         │
│  NOTIFICATIONS               Phase 2    │
│  Encouragement time          9:00 PM >  │
│  Encouragement threshold        60%  >  │
│  Encouragement                   On  >  │
│  Weekly report time          9:00 PM >  │
│  Weekly report                   On  >  │
│  Warning notifications            On  > │
│                                         │
│  ACCOUNT                   (Phase 3)    │
│  Subscription                  Free  >  │
│  Sign in                            >   │
│                                         │
│  STORAGE                   (Phase 3)    │
│  Backend                    Local    >  │
│                                         │
│  ABOUT                                  │
│  Version                        1.0.0   │
│  Feedback                           >   │
│                                         │
│  Recycle Bin                        >   │
└─────────────────────────────────────────┘
```

---

## Sections

### Calendar & Time (Phase 1)

The most culturally sensitive settings. Shown in Phase 1 because `TemporalHelper` reads these from the first day — wrong defaults cause wrong behavior immediately.

**Week starts on** — picker: Sunday / Monday / Friday / Saturday. Default Monday. Middle East users typically select Friday or Saturday.

**Rest days** — multi-select days of week. Default Saturday + Sunday. These are days excluded from certain calculations and shown differently on the performance calendar.

**Day resets at** — time picker for day boundary hour. Default midnight. Night-shift workers or night owls may prefer 3am or 4am.

**Waking hours** — start and end time picker. Default 7am – 10pm. Determines when window warnings and predictive friction notifications fire.

Changes take effect immediately — `TemporalHelper` reads from `UserProfile` on every call.

---

### Notifications (Phase 2)

- **Encouragement time** — when the Path B day celebration notification fires
- **Encouragement threshold** — minimum day score to trigger celebration (default 60%)
- **Encouragement on/off** — disable day celebration entirely
- **Weekly report time** — when the Sunday report notification fires (Pro/Premium only)
- **Weekly report on/off** — disable weekly report (Pro/Premium only)
- **Warning notifications on/off** — global toggle for all window warning notifications

---

### Account (Phase 3)

- Current subscription tier with upgrade option
- Sign in / sign out via Firebase Auth

---

### Storage (Phase 3)

- Current backend: Local or Firebase
- Migration status if in progress

---

### About

- App version
- Feedback link

---

### Recycle Bin

- Soft-deleted commitments — see `recycle_bin.md`

---

## Rules

- Calendar & Time section is Phase 1 — it must be available from first launch
- Every configurable value in the app is exposed here — nothing permanently hardcoded after Phase 2
- Phase 3 sections hidden entirely in Phase 1 and 2 — not shown as disabled, just absent
- Settings changes take effect immediately — no save button needed
- Temporal preference changes propagate instantly via `TemporalHelper` — no app restart needed