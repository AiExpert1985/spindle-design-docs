**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Phase**: 1 (Drift) · Phase 3 (Firestore) **Folder**: Rules

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
- Each user has an independent SQLite file — scales infinitely, zero shared infrastructure

---

## Pro/Premium Tier — Firestore

Drift is abandoned after migration. Firestore with offline persistence always enabled.

- Reads always come from local cache first — fast and free
- Writes go to cache immediately, sync to server when online
- App behaves identically online and offline — no extra code needed
- All analytical reads run against the local cache — Firestore server is never used as an analytics engine

---

## Data Structure — Subcollections per User

All user data stored under the user's document as subcollections:

```
/users/{userId}/commitments/{id}         ← CommitmentDefinition
/users/{userId}/instances/{id}           ← CommitmentInstance
/users/{userId}/logEntries/{id}          ← LogEntry (Activity feature)
/users/{userId}/performanceEntries/{id}  ← CommitmentDailyEntry (Performance feature)
/users/{userId}/weeklyProgress/{id}      ← CommitmentWeeklyProgress (Garment feature)
/users/{userId}/weeklyCups/{id}          ← WeeklyCup (Rewards feature)
/users/{userId}/progression/{id}         ← ProgressionProfile (Progression feature) — one doc
/users/{userId}/aiInsights/{id}          ← AIInsightRecord (AI Insights feature)
/users/{userId}/profile/{id}             ← UserProfile (User feature) — one doc
```

**Why subcollections:**

- User isolation is structural — queries never need a userId filter
- Date range queries on instances and entries use one range filter — zero index read costs
- One security rule covers all collections for all users
- Deleting a user removes all their data automatically

**Drift has no userId field** — local database has one user's data by definition. The repository abstraction handles the path difference transparently.

---

## Repository Ownership

Each subcollection is owned by exactly one repository, which is called only by its own feature's service:

|Subcollection|Repository|Feature|
|---|---|---|
|commitments, instances|CommitmentRepository|Commitment|
|logEntries|ActivityRepository|Activity|
|performanceEntries|PerformanceRepository|Performance|
|weeklyProgress|CommitmentRepository|Garment (via Commitment)|
|weeklyCups|RewardRepository|Rewards|
|progression|ProgressionRepository|Progression|
|aiInsights|AIInsightRepository|AI Insights|
|profile|UserRepository|User|

---

## Streams vs One-Time Reads

|Data|Method|Reason|
|---|---|---|
|Today's instances|Stream|Changes as user logs|
|Active commitment definitions|Stream|Changes on add/edit|
|Garment completion percent|Stream|Changes after midnight update|
|Progression profile|Stream|Changes after Sunday calculation|
|Weekly cups|One-time read|Changes once per week|
|AI insights|One-time read|Changes on demand only|
|Historical instances (calendar)|One-time read|Static|
|Performance entries|One-time read|Static after window close|

Always detach streams when screen is no longer visible. Active listeners consume read quota.

---

## Windowed Fetches — Never Fetch All

Large collections grow continuously. Every fetch must have a time window.

|Use case|Window|
|---|---|
|Dashboard instances|Today only|
|Calendar history|Current month — per navigation|
|Weekly cup calculation|Mon–Sun of target week|
|Performance entries for garment|All entries for one commitment (bounded by commitment age)|
|Analytics / AI|Last 30–60 days, configurable|
|Streak calculation|Last 30 days|
|Bonus trigger evaluation|Last 3 weeks of cup records|

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

---

## Performance at Scale

- Free users: independent SQLite files, zero shared infrastructure, scales infinitely
- Paid users: all reads from local Firestore cache — server reads only on sync
- 100k paid users: subcollection structure means each user's queries only scan their own data