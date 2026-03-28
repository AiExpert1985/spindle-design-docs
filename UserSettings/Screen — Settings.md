**File Name**: screen_settings **Feature**: UserSettings **Phase**: 1 (Calendar & Time) · Phase 2 (Notifications) · Phase 3 (Account, Storage) **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** lets users configure app behavior and manage their account. Temporal and notification preferences write to `UserCoreProfile` through `UserSettingsService`. Encouragement and report preferences write to `UserSettingsProfile` through `UserSettingsService`. Accessed from bottom nav.

---

## Layout

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
│  ACCOUNT                     Phase 3    │
│  Subscription                  Free  >  │
│  Sign in                            >   │
│                                         │
│  STORAGE                     Phase 3    │
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

The most culturally sensitive settings. Available from first launch — `TemporalHelperService` reads these from day one, so wrong defaults cause wrong behavior immediately.

- **Week starts on** — Sunday / Monday / Friday / Saturday. Default Monday.
- **Rest days** — multi-select. Default Saturday + Sunday.
- **Day resets at** — time picker for `dayBoundaryHour`. Default midnight.
- **Waking hours** — start and end time. Default 7am–10pm. Controls notification delivery window.

Changes take effect immediately — `TemporalHelperService` refreshes its cache via `UserCoreService.watchProfile()`.

---

### Notifications (Phase 2)

- **Encouragement time** — when Path B day celebration notification fires
- **Encouragement threshold** — minimum day score to trigger celebration (default 60%)
- **Encouragement on/off** — disable day celebration entirely
- **Weekly report time** — when Sunday report fires (Pro/Premium only)
- **Weekly report on/off** — disable weekly report (Pro/Premium only)
- **Warning notifications on/off** — global toggle for all window warning notifications

---

### Account (Phase 3)

- Current subscription tier with upgrade option — see `component_subscription_tiers`
- Sign in / sign out via Firebase Auth

---

### Storage (Phase 3)

- Current backend: Local or Firebase
- Migration status if in progress — see `service_migration`

---

### About

- App version
- Feedback link

---

### Recycle Bin

- Entry point to `screen_recycle_bin`

---

## Rules

- Calendar & Time section available from Phase 1 — never hidden
- Phase 3 sections absent entirely in Phase 1 and 2 — not shown as disabled
- Settings changes take effect immediately — no save button needed
- Temporal preference changes propagate instantly via `TemporalHelperService` cache refresh

---

## Data Sources

| Data                                       | Source                                                                                                                                                                                                                                                                      |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Temporal preferences                       | `UserCoreService.getTemporalPreferences()` — one-time read on open. UserSettings is the one feature permitted to read directly from `UserCoreService` for temporal preferences since it is the writer of `UserCoreProfile`. All other features use `TemporalHelperService`. |
| Notification and encouragement preferences | `UserSettingsService.getPreferences()` — one-time read on open                                                                                                                                                                                                              |
| All preference updates                     | `UserSettingsService.update*()` functions                                                                                                                                                                                                                                   |