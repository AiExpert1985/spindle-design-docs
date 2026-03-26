**File Name**: userdefaultpreferences **Feature**: UserCore **Phase**: 1 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** starting values for all user-configurable preferences. Written to `UserCoreProfile` and `UserSettingsProfile` once at account creation by `UserCoreService`. Users can change any of them through Settings. The single place that documents what "default" means for every user preference.

Defaults are written once — never re-applied on update or reinstall. Adding a new preference requires: adding the field to the relevant profile model, adding its default here, and reading it in the relevant service.

---

## Regional Onboarding

Temporal preferences vary by region. A one-step prompt on first launch sets the correct starting point without requesting location permission:

```
Where are you based?
[ Middle East ]   [ Europe / Americas ]   [ Other ]
```

Each selection writes the correct temporal defaults to the user profile. "Other" applies the global defaults. Fires once after the first commitment is created.

---

## Temporal Preferences

All time-based logic reads from the user profile. Nothing is hardcoded.

|Preference|Global default|Middle East|Notes|
|---|---|---|---|
|`weekStartDay`|Monday|Saturday|First day of the week|
|`restDays`|Saturday, Sunday|Friday, Saturday|Days excluded from daily scoring average|
|`dayBoundaryHour`|0 (midnight)|0 (midnight)|Hour the calendar day resets|
|`wakingHoursStart`|07:00|07:00|Earliest hour for notification delivery|
|`wakingHoursEnd`|22:00|22:00|Latest hour for notification delivery|

---

## Activity Window Defaults

Applied when a new commitment is created. User can override per commitment in advanced settings.

|Preference|Default|Notes|
|---|---|---|
|`defaultDailyWindowStart`|same as `dayBoundaryHour`|Start of default daily activity window|
|`defaultDailyWindowDuration`|24 hours|Full day|
|`defaultWeeklyWindowStart`|same as `weekStartDay` at `dayBoundaryHour`|Start of weekly window|
|`defaultWeeklyWindowDuration`|7 days|Full week|
|`defaultSpecificDayWindowStart`|same as `wakingHoursStart`|Start on occurrence days|
|`defaultSpecificDayWindowDuration`|16 hours|Waking hours span|

---

## Grace Preferences

|Preference|Default|Notes|
|---|---|---|
|`graceEnabled`|true|Whether the grace re-notification option is offered|
|`gracePeriodMinutes`|15|Duration. Overrides AppConfig default for this user. Zero disables grace.|

---

## Notification Preferences

|Preference|Default|Notes|
|---|---|---|
|`warningNotificationsEnabled`|true|Global toggle for 3/4-elapsed warnings across all commitments|
|`windowCloseNotificationsEnabled`|true|Global toggle for window-close notifications|

---

## Encouragement Preferences

|Preference|Default|Notes|
|---|---|---|
|`celebrationEnabled`|true|Day celebration overlay and notification|
|`celebrationTime`|21:00|Time of day for the celebration notification|
|`celebrationThreshold`|60|Minimum day score percentage to trigger celebration|

---

## Weekly Report Preferences (Pro and Premium only)

|Preference|Default|Notes|
|---|---|---|
|`weeklyReportEnabled`|true|Automatic weekly summary|
|`weeklyReportTime`|21:00|Time of day for the weekly report notification|

---

## Rules

- Defaults written once at profile creation — never re-applied
- Regional defaults set during onboarding — skipping applies global defaults
- Users override any preference in Settings at any time
- System constants that are not user-configurable live in appconfiguration