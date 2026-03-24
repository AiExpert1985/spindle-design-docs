**Created**: 18-Mar-2026 **Modified**: - **Feature**: Achievements **Phase**: 2 (Firestore only — Pro/Premium users are always on Firebase)

**Purpose:** stores and retrieves AchievementRecord entries. Append-only by design — no update or individual delete operations. Abstract interface consistent with all other repositories.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`appendRecord`|AchievementRecord|—|
|`fetchRecords`|limit?, before?: DateTime|List ordered by occurredAt desc|
|`fetchRecordsByCategory`|category, limit?|List for that category ordered desc|
|`hideRecordsForCommitment`|definitionId|— sets hiddenAt on matching records|

---

## Rules

- Never called directly by any feature other than `AchievementsService`
- `appendRecord` is the only write operation — no update, no hard delete
- `fetchRecords` excludes records where `hiddenAt` is set — soft-hidden records never appear in the feed
- `hideRecordsForCommitment` called when a commitment is permanently deleted — `AchievementsService` subscribes to `CommitmentPermanentlyDeletedEvent`
- All fetches are paginated — never fetch all records
- Firestore implementation scopes all queries to `/users/{userId}/achievements/{id}` subcollection

---

## Firestore Path

```
/users/{userId}/achievements/{id}
```

Add to database architecture subcollection list. Covered by existing security rule.