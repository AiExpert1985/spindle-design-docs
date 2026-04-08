**File Name**: error_handling **Phase**: All phases **Feature**: Core**Created**: 26-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** defines how errors are represented and propagated across the app. Every feature follows this convention. No exceptions escape feature boundaries silently.

---

## The Problem

Without a unified error convention, failures scatter across the codebase in inconsistent forms — thrown exceptions, nullable returns, boolean flags, error strings. The caller never knows from the function signature whether it can fail, and if it does, what kind of failure occurred. Silent failures corrupt state without any visible signal.

---

## The Solution — Result Type

Every function that can fail returns `Result<T>`:

```dart
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);
}

class Failure<T> extends Result<T> {
  final AppError error;
  const Failure(this.error);
}
```

This makes failure explicit in the type system. A function that returns `Result<CommitmentInstance>` tells the caller at compile time: this can fail, and you must handle both cases. No surprises, no silent nulls.

The caller pattern-matches exhaustively — the compiler enforces that both cases are handled:

```dart
final result = await commitmentService.create(definition);
switch (result) {
  case Success(:final value) => // use value
  case Failure(:final error) => // handle error
}
```

---

## AppError

`AppError` is the structured failure type. It carries enough information for the logger to record a useful diagnostic without exposing raw exceptions to the rest of the app.

```dart
class AppError {
  final AppErrorType type;
  final String message;     // developer-facing, English only — never shown to users
  final Object? cause;      // original exception, if available
}

enum AppErrorType { network, database, aiApi, validation, unknown }
```

`AppError` lives in the Domain layer — it is a plain value object with no dependencies.

---

## Global Catch

Unhandled exceptions that escape all `Result<T>` wrapping are caught at the app entry point in `main.dart`:

```dart
FlutterError.onError = (details) => logger.error(details.exception, details.stack);
runZonedGuarded(() => runApp(SpindleApp()), (error, stack) => logger.error(error, stack));
```

This is a safety net, not a substitute for `Result<T>`. If an error reaches here, it means a feature boundary was not properly wrapped — that is a bug to fix, not a normal path.

---

## Rules

- Every service function that can fail returns `Result<T>` — no exceptions
- Services never throw raw exceptions across feature boundaries
- `AppError.message` is developer-facing English — never shown directly to users
- Sensitive user data is never included in error messages
- The global catch in `main.dart` is a safety net only — proper wrapping inside features is mandatory
- `Result<T>` and `AppError` live in the Domain layer — they have no dependencies on any feature