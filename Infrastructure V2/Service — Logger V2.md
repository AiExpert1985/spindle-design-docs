**File Name**: logger **Feature**: Infrastructure **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** the single entry point for all diagnostic output in the app. Every feature that needs to record an error, warning, or debug message calls `LoggerService`. Nothing writes to the console, a file, or a remote service directly.

---

## Why a Centralized Logger

Without a centralized logger, diagnostic output scatters — some features use `print()`, others use `debugPrint()`, others are silent. There is no consistent format, no way to control output per environment, and no path to adding persistent or remote logging later.

A centralized abstraction solves all of this. Features call one interface. The implementation behind it is swappable. Adding a new output target — database, remote crash service — requires changing one implementation, not touching every feature.

---

## Log Levels

|Level|Meaning|
|---|---|
|`debug`|Development-only detail. Verbose. Suppressed in production.|
|`info`|Normal significant events — service started, calculation complete.|
|`warning`|Something unexpected happened but the app recovered.|
|`error`|Something failed. Needs attention. Always includes an `AppError`.|

Use levels honestly. Not everything is an error. An `info` log that fires on every tick is noise that buries real warnings.

---

## Interface

```dart
abstract class LoggerService {
  void debug(String message, {String? context});
  void info(String message, {String? context});
  void warning(String message, {String? context});
  void error(AppError error, {StackTrace? stack, String? context});
}
```

`context` is the feature or service name — helps trace where a log originated without parsing stack traces.

---

## Phase 1 Implementation — Console

```dart
class ConsoleLogger implements LoggerService {
  @override
  void debug(String message, {String? context}) =>
      debugPrint('[DEBUG] ${context ?? ''}: $message');

  @override
  void info(String message, {String? context}) =>
      debugPrint('[INFO] ${context ?? ''}: $message');

  @override
  void warning(String message, {String? context}) =>
      debugPrint('[WARNING] ${context ?? ''}: $message');

  @override
  void error(AppError error, {StackTrace? stack, String? context}) =>
      debugPrint('[ERROR] ${context ?? ''}: ${error.message} (${error.type}) caused by ${error.cause}');
}
```

All output goes to `debugPrint()`. Production builds suppress `debug` and `info` entirely.

---

## Later Improvements

**Persistent logger.** Stores warnings and errors in a local database table for post-session review. Requires a repository. Phase 2.

**Remote logger.** Forwards errors to a crash reporting service. Requires network access and user consent. Phase 3.

Both are added by introducing a new `LoggerService` implementation — or a composite that delegates to multiple implementations simultaneously. No feature changes required.

---

## Rules

- Nothing calls `print()`, `debugPrint()`, or any output target directly — always go through `LoggerService`
- `LoggerService` is the abstraction — never reference `ConsoleLogger` or any concrete implementation outside dependency injection
- Log messages are developer-facing English — never shown to users
- Sensitive user data is never included in log messages
- `LoggerService` has no knowledge of any feature — it receives strings and `AppError` values, never domain models