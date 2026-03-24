
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: User
**Phase:** 3

**Purpose:** handles data migration between Drift and Firestore when a user upgrades or cancels their subscription. Ensures no data loss and leaves the app in a consistent state regardless of outcome.

---

## Two Migration Paths

### Upgrade (Free → Pro/Premium)

1. Read all data from Drift (definitions, instances, log entries, weekly cups)
2. Batch write to Firestore in groups of 500 operations
3. Verify all records written successfully
4. Switch active repository to `FirestoreRepository`
5. Clear Drift database
6. Update `storageBackend` in UserProfile via `UserService`

### Cancellation (Pro/Premium → Free)

1. Read all data from Firestore
2. Write to new local Drift database
3. Verify all records written successfully
4. Switch active repository to `DriftRepository`
5. Delete all Firebase user data — nothing remains in the cloud
6. Update `storageBackend` in UserProfile via `UserService`

---

## Atomicity Rule

The old database is kept intact until the new database write is fully verified. If migration fails at any point, the app remains on the original backend with no data loss. Never switch the active repository until the write is confirmed complete.

---

## Functions

### `migrateToFirestore()`

Executes the upgrade migration. Returns a `Result` — success or failure with reason. Called by: subscription upgrade flow.

### `migrateToDrift()`

Executes the cancellation migration. Returns a `Result` — success or failure with reason. Called by: subscription cancellation flow.

### `getMigrationStatus()`

Returns current migration state — not started, in progress, complete, or failed. Used by settings screen to show migration progress indicator.

---

## Rules

- Never exceeds 500 write operations per second to Firestore (500/50/5 rule)
- `storageBackend` on UserProfile is only updated by this service — never by settings directly
- All user Firebase data is deleted on cancellation — nothing remains in the cloud
- Migration failures are logged via `LoggingService` with full context
- Uses `ErrorHandlingService` — returns `Result<T>`, never throws