**File Name**: service_migration **Feature**: User **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** handles data migration between Drift and Firestore when a user upgrades or cancels their subscription. Ensures no data loss and leaves the app in a consistent state regardless of outcome.

---

## Two Migration Paths

### Upgrade (Free → Pro/Premium)

1. Read all data from Drift across all subcollections
2. Batch write to Firestore in groups of 500 operations (500/50/5 rule)
3. Verify all records written successfully
4. Switch active repository to `FirestoreRepository`
5. Clear Drift database
6. Update `storageBackend` to `firebase` in `UserProfile` via `UserService`

### Cancellation (Pro/Premium → Free)

1. Read all data from Firestore
2. Write to new local Drift database
3. Verify all records written successfully
4. Switch active repository to `DriftRepository`
5. Delete all Firebase user data — nothing remains in the cloud
6. Update `storageBackend` to `local` in `UserProfile` via `UserService`

---

## Data Migrated

All subcollections for the user:

```
commitments, instances, logEntries,
garmentProfiles, weeklyProgress,
streakRecords, acceleratorRecords,
achievements, weeklyCups, milestones, rewards,
scoringState, progression,
aiInsights, profile
```

Each subcollection is migrated by reading through its owning repository and writing to the target backend. The repository abstraction handles path differences transparently.

---

## Atomicity Rule

The old database is kept intact until the new database write is fully verified. If migration fails at any point, the app remains on the original backend with no data loss. Never switch the active repository until the write is confirmed complete.

---

## Functions

### `migrateToFirestore()` → Result

Executes the upgrade migration. Returns success or failure with reason. Called by the subscription upgrade flow.

### `migrateToDrift()` → Result

Executes the cancellation migration. Returns success or failure with reason. Called by the subscription cancellation flow.

### `getMigrationStatus()` → MigrationStatus

Returns current migration state — not started, in progress, complete, or failed. Used by the Settings screen to show migration progress.

---

## Rules

- Never exceeds 500 write operations per second to Firestore
- `storageBackend` on `UserProfile` is only updated by this service — never written directly by the Settings screen
- All Firebase user data deleted on cancellation — nothing remains in the cloud
- Migration failures logged via `LoggingService`
- Returns `Result<T>` — never throws

---

## Dependencies

- All feature repositories — reads from source, writes to target
- `UserService` — updates `storageBackend` after successful migration
- `LoggingService` — failure logging