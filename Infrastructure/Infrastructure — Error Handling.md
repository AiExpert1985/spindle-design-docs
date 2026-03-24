
**Created**: 15-Mar-2026
**Modified**: -
**Phase:** 1
**Feature** — Infrastructure

**Purpose:** consistent error handling across all services. Every service function returns a typed result — never throws raw exceptions into the presentation layer. Provides a single place to log, categorize, and surface errors.



---

## The Problem Without This

Without a consistent pattern, each service handles errors differently — some throw exceptions, some return null, some return empty lists. The presentation layer can't know what happened or how to respond. Error logging is scattered.

---

## Result Type

Every service function that can fail returns a `Result<T>`:

```
Result<T>
  → Success<T>  { value: T }
  → Failure<T>  { error: AppError }
```

The presentation layer checks the result type and responds accordingly. It never catches raw exceptions.

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
|network|API timeout, no connection|
|database|Write failed, read returned null unexpectedly|
|ai_api|Anthropic API error, quota exceeded|
|validation|Invalid input, constraint violation|
|unknown|Anything else|

---

## ErrorHandlingService

Single service that services call when an error occurs:

### `handle(error, context)`

Logs the error via `LoggingService`. Categorizes it. Returns an `AppError` the caller wraps in a `Failure` result.

### `isRetryable(error)`

Returns true if the error is transient and worth retrying (network timeout, temporary API error). Returns false for permanent failures (validation errors, missing data).

---

## How Services Use It

```dart
Future<Result<CommitmentInstance>> logEntry(...) async {
  try {
    // write to repository
    return Success(instance);
  } catch (e) {
    final error = errorHandlingService.handle(e, 'logEntry');
    return Failure(error);
  }
}
```

---

## How Presentation Layer Uses It

```dart
final result = await commitmentService.logEntry(...);
result.when(
  success: (instance) => // update UI,
  failure: (error) => // show appropriate message,
);
```

---

## Rules

- Services never throw raw exceptions — always return `Result<T>`
- Error messages are in English — they are for developers, not users
- The presentation layer decides what to show the user — services only categorize
- Sensitive data is never included in error messages
- See `logging_service.md` for how errors are persisted