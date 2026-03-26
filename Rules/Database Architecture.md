**Created**: 15-Mar-2026 **Modified**: 26-Mar-2026 **Phase**: 1 (Drift) · Phase 3 (Firestore) **Folder**: Rules

**Purpose:** defines which database backend is active per tier, how data is structured, how migration works, and the core rule that the rest of the app never knows which backend is in use.

---

## Core Rule — One Backend at a Time

The app uses exactly one database at any moment. Free tier uses Drift. Pro/Premium uses Firestore. They are never active simultaneously.

```
Free:         App → Repository → Drift (SQLite, on device)
Pro/Premium:  App → Repository → Firestore (offline persistence enabled)
```

The repository abstraction means services, providers, and screens never know which backend is active. They call the same interface regardless of tier.

---

## Free Tier — Drift (SQLite)

- All data lives on device only — no internet required, ever
- Full functionality permanently offline
- No Firebase SDK involved at all

---

## Pro/Premium Tier — Firestore

Drift is abandoned after migration. Firestore with offline persistence always enabled.

- Reads always come from local cache first — fast and free
- Writes go to cache immediately, sync to server when online
- App behaves identically online and offline — no extra code needed
- All analytical reads run against the local cache — Firestore server is never used as an analytics engine

---

## Data Structure — Subcollections per User

All user data is stored under the user's document as subcollections. Each feature owns exactly one subcollection, accessed only through its own repository.

**Why subcollections:**

- User isolation is structural — queries never need a `userId` filter
- Date range queries use one range filter — zero composite index costs
- One security rule covers all collections for all users
- Deleting a user removes all their data automatically

For the subcollection-to-repository mapping, see each feature's repository doc.

**Drift has no `userId` field** — local database has one user's data by definition. The repository abstraction handles the path difference transparently.

---

## Repository Ownership

Each subcollection is owned by exactly one repository. That repository is called only by its own feature's service — no other feature touches it directly.

For ownership details, see `feature_dependency_chain` and each feature's repository doc.

---

## Streams vs One-Time Reads

Use a **stream** when the data changes during a session and the UI must stay in sync automatically.

Use a **one-time read** when the data is static for the duration of the screen or is fetched on demand only.

The choice is made per data type in each feature's repository doc.

---

## Windowed Fetches — Never Fetch All

Large collections grow continuously. Every fetch must have a time window or a hard limit. Never fetch an entire collection.

The window size for each use case is defined in the relevant feature's repository or service doc.

---

## Migration

**Upgrade (Free → Pro/Premium):**

1. Batch write all Drift data to Firestore (max 500 operations per batch)
2. Verify migration complete
3. Switch active repository to FirestoreRepository
4. Clear Drift database

**Cancellation (Pro/Premium → Free):**

1. Batch read all Firestore data → write to new Drift database
2. Verify migration complete
3. Switch active repository to DriftRepository
4. Delete all Firebase user data — nothing remains in the cloud

Migration is atomic — old database kept intact until new database write is verified. No data loss on failure.

---

## Security Rule

One rule covers all user data:

```
match /users/{userId}/{collection}/{docId} {
  allow read, write: if request.auth.uid == userId;
}
```

All subcollections covered. No additional rules needed.