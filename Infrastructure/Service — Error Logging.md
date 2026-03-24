
**Created**: 15-Mar-2026
**Modified**: -
**Phase:** 1
**Feature** : Infrastructure

**Purpose:** single entry point for all logs, warnings, and errors in the app. Nothing writes to the console or database directly. Provides a permanent record for debugging and a live stream for development.

---

## What It Does

Every part of the app that needs to log something calls the logging service. The service decides where the output goes — console, local database, or both — depending on the environment and log level.

---

## Log Levels

|Level|Meaning|
|---|---|
|debug|Development-only detail. Verbose.|
|info|Normal significant events (service started, cup earned, report generated).|
|warning|Something unexpected happened but the app recovered.|
|error|Something failed. Needs attention.|

---

## Outputs

**Console** — all levels in debug/development builds. Disabled in production.

**database** — warnings and errors only, in all builds. Stored permanently so issues can be reviewed after the fact. Debug and info are not stored — they would fill the database quickly.

---

## Abstraction

The logging service sits behind an abstract interface. The rest of the app calls this interface only. The implementation handles routing to console and database. In the future, an additional output (remote crash reporting, for example) can be added by changing the implementation — nothing else changes.

---

## Rules

- Nothing writes to `print()`, `debugPrint()`, or any database directly — always go through the logging service
- Log levels are used honestly — don't mark everything as error
- Logged messages are in English regardless of the user's language setting — logs are for developers, not users
- Sensitive user data is never included in log messages