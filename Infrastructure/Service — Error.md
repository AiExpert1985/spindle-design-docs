**File Name**: errorservice **Feature**: Infrastructure **Phase**: 1 **Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** single entry point for all error handling and diagnostic logging in the app. Every service that catches an exception calls `ErrorService`. Every part of the app that needs to log a diagnostic message calls `ErrorService`. Nothing writes to the console or database directly.

Replaces the previously separate `ErrorHandler` and `LoggingService` docs. They were always coupled — `ErrorHandler` called `LoggingService` internally, and every service depended on both. A single service with two responsibilities (handle exceptions, record logs) is simpler with no loss of clarity.

---

## Log Levels

|Level|Meaning|
|---|---|
|`debug`|Development-only detail. Verbose. Never stored.|
|`info`|Normal significant events. Not stored.|
|`warning`|Something unexpected happened but the app recovered. Stored.|
|`error`|Something failed. Needs attention. Stored.|

Warnings and errors are stored permanently in a local database table so issues can be reviewed after the fact. Debug and info are console-only in development builds and suppressed entirely in production.

---

## Error Categories

```
AppError
  type: network | database | ai_api | validation | unknown
  message: String      // developer-facing, English only
  cause: Exception?    // original exception if available
```

|Category|Examples|
|---|---|
|`network`|API timeout, no connection|
|`database`|Write failed, read returned null unexpectedly|
|`ai_api`|Anthropic API error, quota exceeded|
|`validation`|Invalid input, constraint violation|
|`unknown`|Anything else|

---

## Result Type

Every service function that can fail returns `Result<T>`:

```dart
sealed class Result<T>
  Success<T> { value: T }
  Failure<T> { error: AppError }
```

The presentation layer pattern-matches on the result type. It never catches raw exceptions.

---

## Functions

### `handle(exception, context) → AppError`

Categorizes the exception, logs it at `error` level, returns a typed `AppError`. The caller wraps this in a `Failure` result.

```dart
Future<Result<CommitmentInstance>> someOperation() async {
  try {
    final result = await repository.doSomething();
    return Success(result);
  } catch (e) {
    return Failure(errorService.handle(e, 'someOperation'));
  }
}
```

### `log(level, message, {context})`

Records a diagnostic message. Use for non-error observations — service started, scheduled notification fired, weekly cup calculated.

```dart
errorService.log(LogLevel.info, 'Cup calculation complete', context: 'CupService');
```

### `isRetryable(error) → bool`

Returns true for transient failures worth retrying (network timeout, temporary API error). Returns false for permanent failures (validation errors, missing data).

---

## Presentation Layer Pattern

```dart
final result = await commitmentService.createCommitment(definition);
result.when(
  success: (commitment) => // update UI,
  failure: (error) => // show appropriate user-facing message,
);
```

The presentation layer decides what to show the user. Error messages inside `AppError` are English, developer-facing — never shown directly to users.

---

## Abstraction

`ErrorService` sits behind an abstract interface. The rest of the app calls the interface only. The implementation handles routing to console and database. A future output (remote crash reporting) is added by changing the implementation — nothing else changes.

---

## Rules

- Nothing writes to `print()`, `debugPrint()`, or any database directly — always go through `ErrorService`
- Services never throw raw exceptions — always return `Result<T>`
- Log levels used honestly — not everything is an error
- Error messages in English — they are for developers, not users
- Sensitive user data never included in log messages
- No feature knowledge — `ErrorService` receives strings and exceptions, never domain models